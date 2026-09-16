# 검증 방법

이 저장소만으로 확인할 수 있는 것과 없는 것을 먼저 구분한다.

| 확인된다 | 확인되지 않는다 |
|---|---|
| 각 줄이 앞줄에 이어져 있고 중간이 삭제·수정·삽입되지 않았다 | 각 줄이 무엇의 지문인지 (원문 미공개 시) |
| 각 지문이 특정 시점 이전에 존재했다 (비트코인 앵커) | 수익률·성과·실력 |
| 선언일 이후 이 사슬이 유일하게 제시된 기록이다 | 선언일 이전에 다른 사슬이 없었다는 것 |

## 1. 사슬 검사 (설치 불필요)

저장소를 받은 뒤 `python3`로 실행한다. 다른 패키지가 필요 없다.

```python
import json
previous, dates = "0" * 64, []
for index, raw in enumerate(open("commitments.log", encoding="utf-8"), start=1):
    row = json.loads(raw)
    assert row["seq"] == index, f"순번이 어긋난다: {index}"
    assert row["previous_commitment_sha256"] == previous, f"사슬이 끊겼다: seq {index}"
    assert row["date"] not in dates, f"날짜가 중복된다: {row['date']}"
    previous, _ = row["commitment_sha256"], dates.append(row["date"])
print(f"사슬 정상 · {len(dates)}줄 · {dates[0]} ~ {dates[-1]} · 마지막 지문 {previous}")
print("날짜 목록(빠진 날이 있으면 여기서 보인다):", dates)
```

한 줄이라도 바뀌면 그 뒤가 전부 어긋나므로 `AssertionError`로 멈춘다.
날짜가 비어 있는 구간은 그날 기록이 없었다는 뜻이며, 그 자체로 감춰지지 않는다.

## 2. 앵커 검사 (시점)

`ots/` 아래의 `.ots` 파일이 각 줄의 지문에 대한 OpenTimestamps 증명이다.
공식 클라이언트(`pip install opentimestamps-client`)로 확인한다.

```
ots info ots/000001.ots
```

출력의 `File sha256 hash`가 `commitments.log` 첫 줄의 `commitment_sha256`과 같아야 한다.
비트코인 앵커가 아직 대기 중이면 `PendingAttestation(...)`이 보인다. 다음 명령으로 갱신한다.

```
ots upgrade ots/000001.ots && ots info ots/000001.ots
```

`BitcoinBlockHeaderAttestation`이 나타나면 그 블록 시각 이전에 이 지문이 존재했다는 뜻이다.
이 증명은 작성자도 취소할 수 없다.

## 3. 선언 대조

`DECLARATION.md`의 시작 지문이 `commitments.log` 첫 줄과 같은지 본다.
선언 문서 자체의 지문도 함께 앵커돼 있다면 `DECLARATION.md.ots`를 §2와 같은 방법으로 확인한다.

**git 커밋에 찍힌 날짜는 검증에 쓰지 않는다.** 그 값은 작성자의 컴퓨터가 적는다.
시점 근거는 §2의 앵커와 이 저장소 호스트가 기록한 push 이력뿐이다.

## 4. 원문 대조 (원문을 제공받은 경우)

원문 묶음과 무작위 값(salt)을 함께 받았다면, 받은 JSON을 다음 규칙으로 해시해
`commitments.log`의 해당 줄과 일치하는지 본다.

- 정규화: `json.dumps(값, ensure_ascii=False, sort_keys=True, separators=(",", ":"))`의 UTF-8 바이트
- 그 바이트의 SHA-256이 그 날짜의 `commitment_sha256`과 같아야 한다

일치하면 그 원문이 그 시점에 확정돼 있었고 이후 수정되지 않았다는 뜻이다.
일치하지 않으면 원문이 그 줄의 대상이 아니거나 변경된 것이다.

## 문의

검증 중 불일치를 발견하면 저장소 이슈로 알려 주기 바란다. 불일치는 그 자체로 이 기록의 반증이다.
