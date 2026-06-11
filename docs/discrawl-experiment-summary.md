# LLM 기반 Unit Test 유지보수 실험 요약

## 1. 목적

production code가 의도적으로 evolution되는 경우, 이전 behavior를 기준으로 작성된 unit test가 깨질 수 있다. 이 실험은 오픈소스 프로젝트의 작은 변경 사례를 대상으로 그런 상황을 재현하고, LLM이 깨진 test를 새 production behavior에 맞게 유지보수할 수 있는지 확인하기 위한 것이다. 실험의 포인트는 단순히 test를 pass시키는지가 아니라 repair 이후에도 의미 있는 test signal이 남아 있는지 여부다.

예제 프로젝트는 Go 기반의 Discord 데이터 아카이빙 도구인 Discrawl이다. 대상 unit은 `internal/discorddesktop/import.go`와 `internal/discorddesktop/import_pipeline_test.go`에 걸친 Discord desktop cache import 경로다. 특히 cache file의 fingerprint를 재사용할 때, 이전 실행에서 실제로 import된 파일과 route hint가 없어 skip된 파일을 구분하는 behavior를 다룬다.

## 2. Commit Pair

Discrawl importer는 Discord desktop cache file을 스캔할 때, 파일 크기와 수정 시간으로 구성된 fingerprint를 저장해 다음 실행에서 같은 파일을 다시 처리할지 판단한다. 이때 cache file은 두 가지 상태를 가질 수 있다.

- `imported`: 이전 실행에서 파일 내용을 실제로 처리한 상태
- `skipped`: 이전 실행에서 route hint가 없어 fast mode에서 파싱하지 않고 건너뛴 상태

이번 실험에서 사용한 commit 흐름은 다음과 같다.

- base commit `t`: `def592c`
- upstream evolution commit `t+1`: [`055095a recheck skipped cache fingerprints`](https://github.com/openclaw/discrawl/commit/055095aadfc2498b83385210f86b3b0583033f64)
- experiment broken-input commit: `dd90946 test fixture: evolve cache fingerprint reuse behavior`
- Claude repair commit: `f57b527 test: repair cache fast-skip expectation`

`t` 시점의 production behavior는 fast mode에서 cache file fingerprint가 이전 실행과 같으면, 그 파일이 이전에 실제로 import되었는지 여부와 상관없이 `FilesUnchanged`로 처리할 수 있었다. 즉 이전에 route hint가 없어 `skipped`로 기록된 cache file도, fingerprint가 같고 fast mode라면 다시 route hint를 확인하지 않고 unchanged로 분류될 수 있었다.

upstream `t+1`인 [`055095a`](https://github.com/openclaw/discrawl/commit/055095aadfc2498b83385210f86b3b0583033f64)에서는 production behavior와 test expectation이 함께 바뀌었다. production code는 이제 같은 fingerprint가 발견되어도, 이전 상태가 `imported`일 때만 `FilesUnchanged`로 처리한다. 이전 상태가 `skipped`라면 fast mode에서도 route hint를 다시 확인하고, 여전히 route hint가 없으면 `CacheFilesFastSkipped`로 집계한다.

이번 실험에서는 Claude에게 repair해야 할 broken input을 만들기 위해 `055095a`의 production change만 분리해서 `dd90946` 커밋으로 구성했다. 즉 `dd90946`은 새 production behavior는 포함하지만, test expectation은 아직 이전 behavior를 기대하는 상태다. 이후 Claude Code가 test repair를 수행한 결과가 `f57b527` 커밋이다.

핵심 production change는 다음 조건 변경이다.

```go
if !opts.FullCache || isImportedFingerprint(previous) {
```

에서 다음으로 바뀌었다.

```go
if isImportedFingerprint(previous) {
```

즉 `!opts.FullCache`라는 fast mode 조건만으로 unchanged 처리하지 않고, 이전 fingerprint가 실제 import 완료 상태인지 확인하게 된 것이다.

## 3. Legacy Test와 실패 상황

대상 test는 `internal/discorddesktop/import_pipeline_test.go`의 `TestImportFastCacheSkipsUnroutedCacheDataUnlessFullCache`이다. 이 test는 임시 디렉터리에 Discord cache와 비슷한 구조를 만들고, `Cache/Cache_Data/entry_0` 파일에 route hint가 없는 cache-like 데이터를 직접 기록한다.

테스트 데이터 주입은 대략 다음 구조다.

```go
dir := t.TempDir()
cachePath := filepath.Join(dir, "Cache", "Cache_Data")
os.MkdirAll(cachePath, 0o755)
os.WriteFile(filepath.Join(cachePath, "entry_0"), []byte(`...`), 0o600)
```

여기서 `entry_0`에는 message-like JSON data가 들어 있지만, importer의 fast route hint check가 찾는 `/channels/{guildID}/{channelID}` 또는 `/api/v9/channels/{channelID}/messages` 같은 문자열은 없다. 따라서 fast mode에서는 이 cache file을 full parse하지 않고 skip하는 것이 현재 behavior다.

legacy test는 두 번째 fast import에서 같은 fingerprint를 가진 cache file을 `FilesUnchanged`로 기대했다. 그러나 `055095a`에서 도입된 production behavior에서는 이전 상태가 `skipped`이면 unchanged로 처리하지 않고 route hint를 다시 확인한다. route hint가 여전히 없기 때문에 결과는 `CacheFilesFastSkipped`가 된다.

변경 전후 expectation은 다음과 같이 비교할 수 있다.

| Counter | Legacy test expected | Actual value after `055095a` behavior |
| --- | --- | --- |
| `stats.FilesScanned` | `0` | `0` |
| `stats.FilesUnchanged` | `1` | `0` |
| `stats.CacheFilesFastSkipped` | `0` | `1` |

이 실패는 production code의 오류가 아니라 behavior가 바뀌면서 이전 동작을 기준으로 작성된 test expectation이 더 이상 현재 동작과 맞지 않게 된 사례이다. 새 behavior에서는 “파일이 바뀌지 않았다”는 사실과 “파일이 이전 실행에서 실제로 import되었다”는 사실을 구분한다.

## 4. Claude Repair Setup

Claude Code에는 실패 상태의 worktree와 최소 작업 지시만 제공했다. Claude Code에게 작업을 지시한 프롬프트는 다음과 같은 형태였다.

```text
A unit test is failing in this worktree.

Please diagnose the failure and update the test file as appropriate.
Treat the production source files as the current behavior to test;
do not edit production source files.

Command:
go test ./internal/discorddesktop -run TestImportFastCacheSkipsUnroutedCacheDataUnlessFullCache -count=1

Failing output:
[failing test output]
```

production source file을 수정하지 말라는 범위 제약은 제공했다. 이 제약이 없으면 LLM이 test를 새 behavior에 맞게 고치는 대신 production code를 과거 behavior로 되돌려 test를 pass시킬 수 있기 때문이다.

## 5. Claude Repair Result

Claude는 production source file인 `internal/discorddesktop/import.go`는 수정하지 않고, 실패하던 test file인 `internal/discorddesktop/import_pipeline_test.go`만 수정했다. 수정된 test의 핵심은 다음과 같다.

- 두 번째 fast import에서 `FilesUnchanged = 1`을 기대하지 않도록 변경했다.
- 같은 fingerprint라도 이전 상태가 `skipped`이면 route hint를 다시 확인한다는 현재 behavior를 반영했다.
- route hint가 없는 cache file이므로 `CacheFilesFastSkipped = 1`을 기대하도록 변경했다.
- `FilesScanned = 0` expectation은 유지해서 fast mode에서 full scan이 발생하지 않는다는 signal을 남겼다.
- 이후 `FullCache: true` 실행에서 full cache import path가 동작하는 기존 검증 흐름은 유지했다.

Repair 이후 focused test는 통과했다.

```text
go test ./internal/discorddesktop -run TestImportFastCacheSkipsUnroutedCacheDataUnlessFullCache -count=1
```

## 6. 평가 기준과 결과

1. Targeted test가 pass하는가
 - 결과: pass

2. Production source file을 수정하지 않았는가
 - 결과: pass

3. Assertion을 단순 삭제하거나 약화하지 않았는가
 - 결과: pass
 - `FilesScanned`, `FilesUnchanged`, `CacheFilesFastSkipped` counter assertion이 현재 behavior에 맞게 유지되었다.

4. 변경된 production behavior를 검증하는가
 - 결과: pass
 - 이전에 `skipped`로 저장된 cache fingerprint를 같은 fingerprint라는 이유만으로 unchanged 처리하지 않고, route hint를 다시 확인하는 behavior를 검증한다.

5. 의미 있는 test signal이 남아 있는가
 - 결과: pass
 - 수정된 test는 단순히 counter 검증을 제거하지 않았다. 잘못된 구현이 skipped cache file을 unchanged로 다시 분류하면 실패할 수 있는 assertion을 유지한다.

## 7. 해석

이번 실험 사례에서는 LLM에 최소한의 작업 지시와 실패 로그만 제공했을 때도 test repair가 성공했다. Claude는 실패한 assertion만 기계적으로 바꾸는 데 그치지 않고, production behavior가 `FilesUnchanged`와 `CacheFilesFastSkipped`를 구분하도록 바뀌었다는 점을 반영했다.

핵심은 `sameFileFingerprint`가 파일 내용이 이전과 같다는 빠른 추정에는 충분하지만, 그것만으로 해당 파일이 이전 실행에서 실제로 import되었다고 말할 수는 없다는 점이다. 이전 실행에서 route hint가 없어 `skipped`였던 파일은 같은 fingerprint를 유지하더라도 imported data가 아니다. 따라서 새 production behavior에서는 skipped fingerprint를 unchanged로 묻지 않고 route hint를 다시 확인한다.

Claude repair는 이 차이를 test expectation에 반영했다. 결과적으로 repaired test는 현재 production behavior와 정합성을 가지며, 동시에 route hint 없는 cache file이 fast mode에서 잘못 `FilesUnchanged`로 분류되는 회귀를 잡을 수 있는 signal을 유지한다.

다만 이번 결과를 “LLM이 항상 최소 지시만으로 정확한 test repair를 수행한다”는 결론으로 일반화하기는 어렵다. 이 사례는 실패 범위가 작고, 관련 production/test file의 연결이 비교적 직접적이었다. 다음 질문은 어떤 조건에서는 이런 최소 지시만으로 충분하고, 어떤 조건에서는 더 많은 context나 구조화된 repair workflow가 필요한가이다.

## 8. 한계와 후속 실험

이번 실험은 하나의 오픈소스 프로젝트, 하나의 failing test, 하나의 Claude Code 실행을 대상으로 했다. 또한 broken input은 실제 upstream behavior change를 바탕으로 fork branch에서 실험용으로 구성한 것이다. 후속 실험에서는 다음을 추가할 수 있다.

- 작업 지시 방식에 따른 repair 결과 비교
- 여러 LLM model 간 repair 결과 비교
- 여러 failing tests가 동시에 있을 때의 repair quality 평가
- 실제 upstream history에서 test와 production change가 함께 존재하는 사례 추가
- mutation testing 또는 artificial negative case를 통해 repaired test가 potential bugs를 잡아내는지 확인

이번 실험에서는 mutation check를 수행하지 않았다. mutation testing은 repaired test의 강도를 더 엄밀하게 측정하는 별도 축이므로, 본 실험에서는 focused failing test 복구 여부와 production behavior 대비 정합성 평가에 범위를 제한했다.

## 9. 참고 자료

Branches and repair commit:

- Experiment branch: `experiment/discrawl-cache-broken-input`
- Base commit: `def592c`
- Upstream evolution commit: [`055095a recheck skipped cache fingerprints`](https://github.com/openclaw/discrawl/commit/055095aadfc2498b83385210f86b3b0583033f64)
- Experiment broken-input commit: `dd90946 test fixture: evolve cache fingerprint reuse behavior`
- Claude repair commit: `f57b527 test: repair cache fast-skip expectation`

GitHub PR:

- <https://github.com/garibong-labs/discrawl/pull/1>
