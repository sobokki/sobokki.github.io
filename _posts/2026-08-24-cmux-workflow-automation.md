---
title: "cmux를 무작정 쓰다가 작업 환경을 자동화하기까지"
date: 2026-08-24 17:36:46 +0900
categories: [TIL, cmux]
tags: [cmux, codex, claude-code, hooks, developer-tools]
---

## 상황

바이브 코딩을 할 때 주로 Cursor 옆에 터미널을 띄우고 Claude Code나 Codex를 실행했다. 그러다 cmux를 알게 된 뒤부터는 프로젝트마다 워크스페이스를 만들고, 탭과 화면 분할을 이용해 여러 AI 세션을 관리했다.

사용하는 데 큰 문제는 없었지만 정작 cmux가 어떤 구조로 동작하는지는 설명하기 어려웠다.

- 워크스페이스와 분할 화면은 어떤 관계인가?
- cmux의 Hooks와 Skills는 무엇인가?
- 세션 복원은 다른 컴퓨터에서도 이어진다는 뜻인가?
- Claude를 각각 실행하는 것과 Agent Team은 무엇이 다른가?
- 자주 사용하는 화면 구성을 자동으로 열 수 있는가?

단축키를 더 외우기보다 cmux 생태계의 구조부터 짧게 정리해보기로 했다.

## 해결 과정

### 1. 화면 구조부터 구분했다

cmux 화면은 세 가지 단위로 이해하면 간단했다.

```text
Workspace
└── Pane
    └── Surface
```

- **Workspace**: 사이드바에 표시되는 프로젝트 또는 작업 단위
- **Pane**: 화면을 좌우·상하로 나눈 영역
- **Surface**: 각 Pane 안에서 전환하는 터미널·브라우저 탭

따라서 오른쪽 영역을 다시 위아래로 나누는 것은 워크스페이스를 추가하는 것이 아니라, **같은 워크스페이스에 Pane을 추가하는 것**이다.

이 구분을 알고 나니 내가 사용하던 방식도 명확해졌다. 프로젝트별로 Workspace를 나누고, 하나의 프로젝트 안에서는 Pane과 Surface로 프론트엔드·백엔드·AI 세션을 배치하고 있었다.

### 2. cmux와 AI 에이전트의 연결 구조를 이해했다

cmux 자체가 코드를 작성하는 AI는 아니다. Codex나 Claude Code가 작업하고, cmux는 그 작업을 배치하고 관찰하는 역할을 맡는다.

```text
Codex · Claude Code
        │
      Hooks
        │
       cmux
        │
Workspace · Pane · Surface · 알림
```

여기서 자주 헷갈렸던 것이 Hooks와 Skills였다.

- **Hooks**: AI 세션에서 발생한 이벤트를 cmux에 전달한다.
- **Skills**: AI에게 특정 도구를 사용하는 방법과 작업 절차를 알려주는 지침이다.
- **CLI·Socket**: AI나 스크립트가 cmux의 워크스페이스와 화면을 제어하는 통로다.

사용 중이던 cmux 버전에서는 다음 명령을 지원하지 않았다.

```shell
cmux hooks setup
```

cmux를 업데이트한 뒤 다시 실행하자 Codex용 Hooks가 설치됐다.

```shell
cmux version
# cmux 0.64.22

cmux hooks setup
```

Hooks 설치 후에는 Codex의 시작·중지·도구 사용·권한 요청 같은 상태를 cmux가 전달받을 수 있게 됐다.

이 기능은 세션을 다른 컴퓨터로 동기화하는 기능과는 다르다. cmux의 워크스페이스 복원은 기본적으로 **같은 컴퓨터에 남아 있는 로컬 상태와 프로세스**를 대상으로 한다.

### 3. Agent Team은 항상 더 좋은 방식이 아니었다

여러 Claude를 각각 실행하면 사람이 직접 작업을 나누고 결과를 확인한다. 반면 Claude Agent Team에서는 대표 Claude가 작업 목록을 만들고 다른 Claude에게 일을 배정하고 취합한다.

```text
개별 세션: 내가 각 Claude를 조율
Agent Team: 대표 Claude가 다른 Claude를 조율
```

Agent Team이 적합한 작업은 하나의 목표가 독립적인 작업으로 명확하게 나뉘는 경우다.

```text
결제 기능
├── 프론트엔드
├── 백엔드
└── 테스트
```

반대로 하나의 파일을 함께 수정하거나 순서대로 진행해야 하는 작업은 조율 비용과 충돌 가능성만 커질 수 있다. Agent Team은 아직 실험 기능이며, 여러 컨텍스트가 동시에 동작하므로 토큰 사용량도 증가한다.

결국 평소에는 기존처럼 독립 세션을 사용하고, 규모가 큰 병렬 작업에만 Team을 선택적으로 사용하는 것이 내 작업 방식에 더 적합했다.

- [cmux의 Claude Code Teams 연동](https://cmux.com/docs/agent-integrations/claude-code-teams)
- [Claude Code Agent Teams 문서](https://code.claude.com/docs/en/agent-teams)

### 4. 자주 쓰는 화면 구성을 필요할 때만 열도록 했다

내가 자주 사용하는 구성은 다음과 같다.

```text
┌─────────────────┬─────────────────┐
│                 │ Frontend        │
│ Codex           ├─────────────────┤
│                 │ Backend         │
└─────────────────┴─────────────────┘
```

Codex에서는 전체 흐름과 작업 순서를 관리하고, 오른쪽 위아래 터미널은 프론트엔드와 백엔드 작업에 사용한다.

하지만 모든 프로젝트를 항상 이 구조로 열고 싶지는 않았다. 그래서 새 워크스페이스의 기본값으로 지정하지 않고, 필요할 때 명령 팔레트에서 실행하는 `cmux.json` 명령으로 등록했다.

전역 설정 파일의 위치는 다음과 같다.

```text
~/.config/cmux/cmux.json
```

등록한 레이아웃의 핵심 구조는 이렇다.

```json
{
  "commands": [
    {
      "name": "Codex + Frontend + Backend",
      "workspace": {
        "name": "Fullstack Flow",
        "cwd": ".",
        "layout": {
          "direction": "horizontal",
          "split": 0.5,
          "children": [
            {
              "pane": {
                "surfaces": [
                  {
                    "type": "terminal",
                    "name": "Codex",
                    "command": "codex",
                    "focus": true
                  }
                ]
              }
            },
            {
              "direction": "vertical",
              "split": 0.5,
              "children": [
                {
                  "pane": {
                    "surfaces": [
                      {
                        "type": "terminal",
                        "name": "Frontend"
                      }
                    ]
                  }
                },
                {
                  "pane": {
                    "surfaces": [
                      {
                        "type": "terminal",
                        "name": "Backend"
                      }
                    ]
                  }
                }
              ]
            }
          ]
        }
      }
    }
  ]
}
```

이제 프로젝트에서 `⌘⇧P`를 누르고 `Codex + Frontend + Backend`를 선택할 때만 새로운 3분할 워크스페이스가 만들어진다.

프론트엔드와 백엔드 터미널에는 실행 명령을 고정하지 않았다. 프로젝트에 따라 사용하는 명령과 AI가 달라질 수 있기 때문이다.

- [cmux 설정 문서](https://cmux.com/docs/configuration)
- [cmux Custom Commands 문서](https://cmux.com/docs/custom-commands)

## 배운 것

1. **cmux는 AI가 아니라 AI 작업 환경을 구성하는 조율 계층이다.** 실제 작업은 Codex와 Claude Code가 하고, cmux는 세션과 화면·상태·알림을 관리한다.
2. **Workspace, Pane, Surface는 역할이 다르다.** 프로젝트는 Workspace로, 분할 화면은 Pane으로, 각 영역의 탭은 Surface로 구분하면 된다.
3. **Hooks와 Skills는 다르다.** Hooks는 이벤트 전달 장치이고, Skills는 AI가 따라야 할 작업 지침이다.
4. **세션 복원과 다른 컴퓨터로의 동기화는 같은 기능이 아니다.** 로컬에서 다시 열리는 것만으로 클라우드 동기화가 된 것은 아니다.
5. **Agent Team은 상위 버전의 사용법이 아니라 선택 가능한 조율 방식이다.** 독립성이 높은 병렬 작업에서만 장점이 커진다.
6. **자동화는 기본값보다 필요할 때 호출하는 방식이 더 잘 맞을 수 있다.** 자주 쓰는 구조라도 모든 프로젝트에 강제하지 않고 명령으로 만들어두면 유연성을 유지할 수 있다.

cmux를 처음에는 여러 터미널을 한곳에 모아주는 도구로만 생각했다. 구조를 알고 나니 실제 핵심은 **여러 AI 작업을 배치하고, 상태를 전달받고, 반복되는 작업 환경을 재현하는 것**에 있었다.

Cursor와 Chrome처럼 이미 편하게 사용하는 도구를 모두 cmux 안으로 옮길 필요도 없었다. cmux가 잘하는 조율과 복원은 cmux에 맡기고, 코드와 Git Graph는 Cursor에서, 문서는 Chrome에서 보는 현재 방식이 내게는 가장 자연스러웠다.
