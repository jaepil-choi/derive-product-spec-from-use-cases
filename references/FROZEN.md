# Frozen References

이 디렉터리의 하위 레포들은 **git submodule 이 아니라 특정 커밋에서 떠온 정적 스냅샷**입니다.
자체 harness 를 설계하기 위한 읽기 전용 참고 자료이며, 상류(upstream)를 추적하지 않습니다.

- 스냅샷 시점: 2026-08-17
- 각 스냅샷은 원본 레포의 `.git` 을 제거한 평범한 파일들로 이 레포에 그대로 커밋되어 있습니다.
- `.gitattributes` 의 `export-ignore` 로 릴리스 아카이브(`git archive`)에서는 제외됩니다.

## 스냅샷 목록

| 경로 | 원본 | 고정 커밋 | 태그 | 커밋 날짜 |
| --- | --- | --- | --- | --- |
| `references/ouroboros` | https://github.com/Q00/ouroboros.git | `fc774f7b962964f8ed7e559e1c4618f4699a65f7` | `v0.51.4` | 2026-08-14 |
| `references/gajae-code` | https://github.com/Yeachan-Heo/gajae-code.git | `5666472818b71a1c37615408d9b4d3b5a77b7fa3` | `v0.13.2-1-g566647281` | 2026-08-13 |

## 규칙

- **이 디렉터리의 파일은 수정하지 않습니다.** 참고해서 얻은 결론이나 파생 설계는 이 레포의 `SKILL.md`, `docs/`, `agents/` 에 직접 작성합니다.
- 상류의 최신 변경이 필요해지면 스냅샷을 갱신하지 말고, 필요한 부분만 확인한 뒤 새 스냅샷으로 교체할지 별도로 판단합니다.
- 각 스냅샷은 원본 레포의 라이선스를 따릅니다 (`references/*/LICENSE` 참고).

## 스냅샷 갱신 방법

특정 스냅샷을 다른 커밋으로 교체할 때:

```bash
REPO=https://github.com/Q00/ouroboros.git
SHA=<new-commit-sha>
DEST=references/ouroboros

rm -rf "$DEST" /tmp/frozen-snap
git clone "$REPO" /tmp/frozen-snap
git -C /tmp/frozen-snap checkout "$SHA"
rm -rf /tmp/frozen-snap/.git
mv /tmp/frozen-snap "$DEST"
```

교체 후 위 표의 커밋 SHA / 태그 / 날짜를 반드시 갱신합니다.
