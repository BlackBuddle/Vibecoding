# 자동완성 기능 개발 기록

> 이 문서는 AI(와 사람)가 이 기능이 **왜 이렇게 만들어졌는지** 읽고 이어서 작업할 수 있도록 남긴 기록이다.
> 날짜: 2026-10-01. 작업 환경: Windows 10, Python 3.14.7, PyQt5 5.15.11 (Qt 5.15.2), jedi 0.20.0 / parso 0.8.7.

## 1. 한 줄 요약

`Main.py` 편집기(`CodeEditor`)에 **자동완성**을 추가했다. 코드를 치면 후보 팝업이 뜨고 **Tab**으로 완성된다.
`self.` 뒤에는 `.ui`의 위젯 이름, `self.위젯.` 뒤에는 그 위젯 클래스의 메서드·시그널, 그 밖에는 `jedi`가 PyQt5 클래스·변수·키워드를 준다.

## 2. 사용자 요구와 결정 (사용자가 직접 정한 것)

| 항목 | 결정 | 근거 |
|---|---|---|
| 기능 필요성 | 편집기에 자동완성이 없어서 넣기로 함 | `editor.py`는 `QPlainTextEdit` + 줄 번호 + 하이라이트 + hover뿐이었음 |
| 후보 범위 | 키워드·현재 파일 단어 / `self.` 뒤 `.ui` 위젯 이름 / `self.위젯.` 뒤 메서드·시그널 / PyQt5 클래스·모듈 이름 **전부** | 사용자가 4개 모두 선택, "VS Code 정도 수준으로는 어렵겠지?"라고 물음 |
| VS Code 수준 | **완전 동일은 불가, 가깝게**: 후보 목록·종류 표시까지. 자동 import·깊은 타입 추론·시그니처 팝업은 제외 | 아래 4장 참고 |
| 수락 키 | **Tab만** 수락. Enter는 수락하지 않고 팝업만 닫고 줄바꿈 | 사용자: "탭을 누르면 자동완성되도록". 단어를 다 친 뒤 Enter가 줄바꿈이 안 되는 불편을 피함 |
| 새 의존성 | `jedi` 설치 승인 | 설계 안내 때 "jedi 설치도 승인에 포함"이라고 밝히고 승인받음 |
| 진행 방식 | bounded(기존 편집기에 기능 추가) → 채팅에서 짧은 설계 → 승인 → TDD | `superpowers:brainstorming`, `superpowers:test-driven-development` |

## 3. 설계

**방식**: `jedi`가 일반 파이썬 완성을 맡고, `.ui` 정보로 만든 후보를 그 앞에 붙인다.

| 입력 | 후보 | 출처 |
|---|---|---|
| `de`, `imp` | 키워드, 현재 파일의 변수·함수 | jedi (없으면 자체 키워드 + 파일 단어) |
| `QtWi`, `QPu`, `Qt.Al` | PyQt5 모듈·클래스·상수 | jedi (PyQt5 `.pyi` 스텁) |
| `self.pu` | `.ui` 위젯 이름 (`위젯` 표시) | `.ui` 모델 (`node.name`) |
| `self.pushButton.` | 그 위젯 클래스의 시그널·메서드 | `.ui` 모델 (`node.cls`) + PyQt5 클래스 `dir()` |

`self.위젯.`을 jedi에 맡기지 않은 이유: `uic.loadUi()`로 불러오는 코드는 위젯의 타입이 런타임에야 정해져 jedi가 모른다.
시그널과 메서드는 `isinstance(attr, QtCore.pyqtSignal)`로 구분한다 (시그널을 먼저 보여 줌).

**동작 규칙**
- 식별자를 **2글자 이상** 치거나 `.`을 치면 150ms 뒤에 요청한다. `Ctrl+Space`는 글자 수와 상관없이 즉시 요청한다.
- 주석과 문자열 안에서는 후보를 내지 않는다.
- 이미 완전히 친 단어와 같은 후보는 목록에서 뺀다 (Tab이 아무 일도 안 하는 팝업을 막기 위해).
- ↑↓ 선택(끝에서 반대쪽으로 돌아감), Esc 닫기, 방향키·클릭·포커스 아웃·그 밖의 키는 팝업을 닫는다.
- 수락은 `beginEditBlock`/`endEditBlock`으로 감싸 **Ctrl+Z 한 번**에 되돌아간다.
- `jedi`가 없으면 에러 없이 키워드·파일 단어·`.ui` 후보만 나온다 (`Completer(use_jedi=...)`, `has_jedi`).

## 4. 실측 결과와 그에 따른 설계 변경

설계 단계에서는 "느리면 스레드로"라고 위험으로만 적었는데, 확인해 보니 **처음부터 스레드가 필요**했다.

| 측정 (이 PC) | 시간 |
|---|---|
| jedi 첫 호출 (PyQt5 스텁 파싱) | 2.4초 ~ 5초 |
| `jedi.preload_module(QtWidgets, QtCore, QtGui)` | 약 1.4초 |
| 그 뒤 첫 로컬 변수 멤버 완성 | 약 1.3초 |
| 이후 호출 | 40 ~ 290ms |

→ jedi 호출은 **백그라운드 스레드 하나**(`_CompletionWorker`)에서 하고, 앱이 편집기를 만들 때 `warmup()`으로 미리 데운다.
최신 요청만 처리하고(이전 요청은 버림), 결과는 시그널로 UI 스레드에 돌려준다.
요청 번호(`_req_id`)와 요청 당시 커서 위치(`_asked_at`)가 현재와 다르면 늦게 온 결과를 버린다.

**Tab이 글자를 망가뜨리는 문제를 막은 방법**: 팝업이 `pu` 기준으로 떠 있는데 사용자가 빠르게 `sh`를 더 치고 Tab을 누르면, 새 답이 오기 전이라 `sh`(2글자)가 후보로 덮어써져 `self.puspushButton`이 될 수 있다.
→ 글자를 칠 때마다 이미 떠 있는 목록을 **그 자리에서 좁히고**(`narrow`) `_prefix_len`도 같이 갱신한다. 일치하는 게 없으면 팝업을 닫는다. (테스트 `test_tab_right_after_fast_typing_does_not_corrupt_the_text`)

## 5. 파일 지도

| 파일 | 역할 |
|---|---|
| `studyhelper/completer.py` (신규) | 후보 만들기. Qt 이벤트 루프 없이 테스트 가능한 순수 로직. `Completer.complete(source, line, col, force)` → `(items, prefix_len)`, 줄·열은 0부터 |
| `studyhelper/editor.py` | `_CompletionWorker`(스레드), `_CompletionPopup`/`_KindDelegate`(목록 창), `CodeEditor`의 키 처리·수락. `set_widgets({이름: 클래스})` 추가 |
| `studyhelper/mainwindow.py` | `set_names(...)` 3곳을 `set_widgets(...)`로 교체 (`.ui`의 `node.name → node.cls` 전달) |
| `tests/test_completer.py` | 후보 로직 27개 (`.ui` 후보, 키워드·단어, 주석·문자열, jedi) |
| `tests/test_editor_autocomplete.py` | 편집기 동작 18개 (오프스크린 Qt에서 실제 키 입력) |
| `requirements.txt`, `README.md` | `jedi>=0.19` 추가, 기능 설명 |

## 6. 테스트

```bash
python -m unittest discover -s tests -t .
```

- 결과: **45개 통과** (후보 로직 27 + 편집기 동작 18), 약 7초.
- `pytest`는 이 PC에 없어서 표준 `unittest`를 썼다.
- 실제 앱 창 확인(오프스크린 `MainWindow`에 임시 `.ui` + `Main.py`를 열고 입력): `self.pu` → Tab → `self.pushButton`, 이어서 `.cl` → Tab → `self.pushButton.clicked` 확인.
  이 확인은 앱 설정이 레지스트리에 남지 않도록 `QSettings`를 임시 ini로 돌려서 했다.
- TDD 순서: 테스트 먼저 작성 → 실패 확인(모듈/메서드 없음) → 구현 → 통과. 테스트 작성 중 틀린 부분 2곳(불필요한 `setUp()` 재호출, Backspace 횟수)은 실행 전에 고쳤다.

## 7. 함정 (다음 사람이 같은 데서 막히지 않도록)

1. **`QTest.keyClicks(widget, "\n")`는 이 환경에서 프로세스를 죽인다.** 종료 코드 `0xC0000409`, 메시지 없음. 순정 `QPlainTextEdit`에서도 재현되므로 구현 버그가 아니다 (PyQt5 5.15.11 + Python 3.14). 줄바꿈은 `QTest.keyClick(w, Qt.Key_Return)`로 보낸다.
2. **PyQt5는 슬롯·이벤트 핸들러 안의 처리 안 된 파이썬 예외에서 프로세스를 중단**시킨다. 이때 `0xC0000409`가 나온다. `keyPressEvent` 안 코드를 고칠 때 예외가 새지 않게 한다.
3. **저장소 파일은 CRLF**다 (`core.autocrlf=true`, `git ls-files --eol`에서 `i/crlf w/crlf`). Git Bash의 `sed -i`로 고치면 CR이 사라져 diff가 파일 전체로 부풀었다 (`mainwindow.py` 3줄 수정이 2,204줄 변경으로 보임).
   복구: `sed -i 's/$/\r/' 파일`. 고친 뒤 `git diff --stat`으로 의도한 줄만 바뀌었는지 확인한다. 편집은 Edit 도구를 쓰면 줄바꿈이 유지된다.
4. 이 PC는 `QT_QPA_PLATFORM_PLUGIN_PATH`가 낡은 값일 수 있어 `run.py`가 보정한다. 테스트는 `QT_QPA_PLATFORM=offscreen`으로 돌린다.
5. `jedi`는 스레드에 안전하지 않다고 보고 **스레드 하나**에서만 호출한다. 호출을 병렬화하지 말 것.
6. 지금은 `.py` 파일 경로를 jedi에 넘기지 않아 `from gui import Ui_MainWindow` 같은 이웃 파일 import는 해석하지 못한다 (아래 8장).

## 8. 남은 일 (이번 커밋에 없음)

- **시그니처 팝업**(함수 괄호 안 인자 안내): 일부러 제외.
- **`self.위젯.시그널.` 뒤** (`.connect`, `.emit`): 후보가 안 나온다. `loadUi` 방식 코드는 jedi가 타입을 모른다. 시그널 객체용 후보(`connect`, `disconnect`, `emit`)를 `.ui` 후보 쪽에 추가하면 된다.
- **이웃 파일 해석**: `jedi.Script(source, path=...)`와 `jedi.Project`를 쓰면 `gui.py`의 `Ui_*` 클래스를 해석할 수 있다.
- 큰 파일에서 느리면 `toPlainText()` 전달 방식을 줄이는 것도 고려.

### 같은 날 추가로 요청받았지만 **아직 설계·승인 전**인 기능 (구현하지 않음)

1. **학습할 폴더 경로를 미리 넣어두는 기능** — 어떤 방식인지(기본 폴더 하나인지, 여러 개 목록인지, 어디에 보이는지) 아직 정해지지 않았다.
2. **파일 열기를 하면 이전에 열었던 디렉토리부터 보이게 하기** — `MainWindow.open_dialog()` (`studyhelper/mainwindow.py:667`) 쪽. 앱 설정은 `QSettings("PyQtStudyHelper", "PyQtStudyHelper")`에 저장된다 (`mainwindow.py:111`).

이 둘은 자동완성과 별개의 작업이라 따로 설계 확인을 받고 진행한다.

## 9. 작업 환경 메모 (참고)

- 저장소는 처음에 `D:\git_down\Vibecoding`에 클론했다가, 사용자가 `C:\git_down\pyQt`에 직접 클론하길 원해 그쪽에 다시 클론했다. D 쪽 클론은 삭제했다.
  `D:\git_down` 빈 폴더는 도구가 삭제를 막아 남아 있다.
- `git`에 커밋 작성자(`user.name`, `user.email`)가 설정돼 있지 않았다. 이 저장소의 기존 커밋 작성자는 `Dawnilove`다.
- 저장소에 `.gitignore`가 없어 `__pycache__/`가 추적 안 된 파일로 잡힌다. 커밋할 때 파일을 하나씩 지정해서 올렸다.
