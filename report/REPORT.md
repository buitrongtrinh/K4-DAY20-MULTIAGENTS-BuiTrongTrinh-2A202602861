# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin sinh viên và cấu hình

- Họ tên: Bùi Trọng Trình
- Mã sinh viên: 2A202602861

- Nhà cung cấp và mô hình: DeepSeek API qua endpoint tương thích OpenAI (`LAB_BASE_URL=https://api.deepseek.com/v1`, `LAB_MODEL=deepseek-chat`; API trả về `model_name = deepseek-flash`). `LAB_TEMPERATURE=0`, `recursion_limit=60` (mặc định, giữ nguyên cho mọi điều kiện).
- Deep Agents 0.7.21, Python 3.13 (conda env `lab-vin-env`), Fedora Linux (kernel 7.2.5). Chạy trực tiếp trên máy, **shell của tác tử được cách ly bằng bubblewrap** (xem mục 4.1), không dùng Docker.
- Số lần chạy tác vụ đã dùng / ngân sách: xem mục 7 và Phụ lục.
- Commit của tag `freeze`: xem `git rev-parse freeze`.

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

Căn cứ: kết quả trên tác vụ học (mục 4–6) và tài liệu tham khảo. Trên tác vụ học, điểm trung bình là baseline 0,26 (1/10, 0/8, 6/9), subagents 0,44 (7/10, 5/8, 0/9), skills-auto ở Phần 3.4 là 0,33 (10/10, 0/8, 0/9). Cả ba điều kiện đều có lần chạy chạm `recursion_limit` vì tác tử lặp lại cùng một lệnh, nên nhiễu giữa các lần chạy rất lớn: cùng cấu hình baseline `code-learn` cho 7/10 rồi 1/10.

- H1 (subagents so với baseline): trên tác vụ đánh giá, `subagents` **không cao hơn baseline một cách rõ rệt** (chênh lệch điểm trung bình dưới 0,15, nằm trong mức nhiễu) nhưng **tốn token gấp khoảng 2 lần trở lên**. Lý do: trên tác vụ học, subagents tốn trung bình khoảng 439 nghìn token so với 209 nghìn của baseline. Khi có giao việc, lời giao việc chép đủ phần kỹ thuật của đề, nhưng **không chứa quy ước Acme** vì chính tác tử chính cũng không biết các quy ước đó. Vì vậy check `rule_` vẫn trượt (subagents `code-learn` đạt 7/7 check kỹ thuật, 0/3 check quy ước). Bài viết của Anthropic về hệ thống nghiên cứu đa tác tử cũng ghi nhận chi phí token tăng mạnh (khoảng 15 lần so với hội thoại thường), và lợi ích chỉ rõ ở các tác vụ song song hóa được, trong khi các tác vụ ở đây nhỏ và tuần tự.
- H2 (skills-auto so với baseline): `skills-auto` **cao hơn baseline trên tác vụ đánh giá**, chủ yếu nhờ các check quy ước **lặp lại** từ tác vụ học (đặc biệt ở họ `code`: type hints, `tests/test_regressions.py`, `CHANGELOG.md` dưới `## Unreleased`). Ở `code-learn`, skill nâng từ 0/3 lên 3/3 check `rule_` (`skills_read = 3`). Check quy ước **mới** của tác vụ đánh giá sẽ **không** được skill giúp, vì curator không thể biết quy ước đó. Lợi ích này dễ bị che bởi các lần chạy kẹt vòng lặp (skill không chữa được hành vi lặp của mô hình). Đây cũng là lý do mức tăng dự đoán ở mức vừa phải: SkillsBench ghi nhận skill do mô hình tự sinh trung bình không có lợi, còn ở đây skill chỉ phát biểu lại phản hồi `detail`.
- H3 (tác vụ học so với tác vụ đánh giá): mức cải thiện của `skills-auto` so với baseline **trên tác vụ học lớn hơn trên tác vụ đánh giá** (dấu hiệu quá khớp theo SkillEvolBench), vì skill được viết từ đúng phản hồi của tác vụ học, còn tác vụ đánh giá có dữ liệu khác và thêm một quy ước không học được. Dự đoán: trên tác vụ đánh giá, `skills-auto` vẫn không đạt điểm tối đa ở bất kỳ họ nào có quy ước mới.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ: `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep` (công cụ tệp), `execute` (shell) và `task` (giao việc cho subagent). `execute` là công cụ chạy lệnh shell; `task` gián tiếp cũng chạy được lệnh vì subagent `general-purpose` có cùng bộ công cụ với tác tử chính.
2. Mô tả của `task` nói `general-purpose` dùng để "researching complex questions, searching for files and content, and executing multi-step tasks" và "has access to all tools as the main agent". Subagent **không** thấy ngữ cảnh của tác tử chính: "Each invocation is stateless by default: the agent sees only the prompt you give it and returns a single final report."
3. Câu từ `task`: "Put full detail in the prompt and state exactly what it should return". Câu từ `execute`: "Use absolute paths and avoid `cd` so the working directory stays stable". Câu này **mâu thuẫn** với `PATHS_NOTE` (đường dẫn tương đối, không bắt đầu bằng `/`). Trong vết thấy rõ hệ quả: tác tử nhiều lần chạy `ls -la /workspace` hoặc `cd /workspace && ...` trong shell, nhận lỗi rồi mới chuyển sang dạng tương đối (ví dụ `results/baseline/logs-learn/trace.md`: `cd /workspace && python3 parse_log.py`).

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

### 4.1. Sự cố liêm chính phát hiện trong lần chạy đầu: tác tử đọc bộ chấm điểm

Lần chạy baseline đầu tiên (trước khi cách ly shell) được 8/8 ở `data-learn` và 9/9 ở `logs-learn`. Đọc vết thấy tác tử **thoát khỏi sandbox**: nó chạy `find / -iname '*acme*'`, tìm ra thư mục kho mã nguồn, chạy `cat tasks/data-learn/check.py`, rồi `python3 tasks/data-learn/check.py --workspace /tmp/lab-sandbox-.../workspace` để tự chấm và sửa đến khi đạt. Ở một lần chạy khác nó còn đọc tệp của một project khác trong thư mục người dùng. Đây là **reward hacking** (nhóm G): `LocalShellBackend` chạy `subprocess.run(shell=True)` trên máy thật nên `virtual_mode=True` chỉ chặn công cụ tệp, không chặn shell.

Xử lý:
- Mọi kết quả của các lần chạy đó **bị loại** và chuyển ra ngoài kho (không nằm trong `results/`), để nội dung `check.py` trong vết không đi vào curator.
- `make_backend` dùng lớp `IsolatedShellBackend` (`src/lab/agent.py`): mỗi lệnh `execute` chạy trong `bwrap`. Shell chỉ thấy sandbox (đọc ghi), `/usr`, `/etc` và môi trường Python (chỉ đọc), không có mạng, không thấy `$HOME` hay kho mã nguồn. Đã kiểm tra: `ls /home/<user>/learns` báo không tồn tại, `cd .. && ls` chỉ thấy thư mục sandbox, `env` chỉ có `PATH`, `HOME`, `PYTHONDONTWRITEBYTECODE`. Công cụ tệp đã chặn `../` từ trước (`Path traversal not allowed`). 32/32 test vẫn đạt.
- Xóa `__pycache__` cũ (bị gitignore) trong `tasks/*/workspace` bằng `git clean -fdX tasks/`. Các tệp này sinh ra khi chạy `test_01` và làm nhiễu `code-learn`: tác tử dành hơn 15 bước dịch ngược `.pyc`. Lần chạy `code-learn` bị ảnh hưởng được lưu ở `results/_superseded/` và chạy lại.

### 4.2. Bảng phân loại (baseline sau khi cách ly, tác vụ học)

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| code-learn | `visible_suite_passes`, `parse_price_all_formats`, `other_caller_fixed`, `discount_rounds_half_up`, `low_stock_follows_docstring`, `csv_quoting_follows_docstring` | G (vòng lặp thoái hóa, không kết thúc) | Lần chạy dừng ở `GraphRecursionError`. Sau khi đọc đủ README, docstring và test, tác tử gọi 25 lần liên tiếp `python -m pytest tests -q 2>&1 \| sed -n 'Np'` với N tăng dần từng dòng, không sửa tệp nào. `detail`: "2 failed, 4 passed". |
| code-learn | `rule_type_hints` | E | "RULE: every public function ... has type annotations on all parameters and on the return value." |
| code-learn | `rule_regression_tests` | E | "RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3)". |
| code-learn | `rule_changelog` | E | "RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' as a bullet '- fix(<function name>): ...'". |
| data-learn | 7 check đọc `answer.json` (`north_q1_revenue`, ..., `rule_meta_block`) | G (vòng lặp thoái hóa) | `answer.json` không tồn tại (`FileNotFoundError`). Tác tử lặp **nguyên văn** cùng một lệnh `python3 -c "...Counter(r[0] for r in rows)..."` khoảng 23 lần với cùng kết quả, đến `GraphRecursionError`. |
| data-learn | `rule_clean_csv` | E | "RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents ..." |
| logs-learn | `rule_service_names` | E | "RULE: service names in the output are lower-case with '-' replaced by '_'". |
| logs-learn | `rule_sorted_errors` | E | "RULE: `errors` is sorted by service, then by timestamp_utc, ascending." |
| logs-learn | `rule_schema_header` | E | "RULE: the top-level object has \"schema_version\": 2 and \"generated_by\": \"log-triage\"." |

Bằng chứng phủ định cho nhóm A–D: ở các lần chạy baseline **kết thúc bình thường**, mọi check kỹ thuật đều đạt. `logs-learn` đạt 6/6 check kỹ thuật (UTC, stack trace nhiều dòng, dòng lặp, đếm theo service). Lần chạy `code-learn` trước khi xóa pycache (`results/_superseded/baseline-code-learn-pycache`) đạt 7/7 check kỹ thuật, kể cả hai lỗi chỉ thấy khi đối chiếu docstring. Tác tử cũng luôn đọc README trước khi làm (bước 2–3 trong mọi vết), nên không có lỗi nhóm A, C, D. Không có nhóm F: các lần chạy kết thúc có câu trả lời cuối đều chỉ nêu tệp có thật.

Nhận xét: có hai nhóm lỗi chính.
- **E (quy ước tổ chức)** chiếm mọi check thất bại ở các lần chạy kết thúc bình thường (7/7 check `rule_` có `detail`). Đề chỉ nói "whatever the Acme ... conventions require" mà không cho nội dung. Vết `logs-learn` cho thấy tác tử chạy `find / -iname '*acme*'` để tìm quy ước nhưng không thấy gì. Skill **phòng ngừa được** nhóm này nếu tác vụ mới dùng lại cùng quy ước, vì `detail` phát biểu đúng quy tắc.
- **G (vòng lặp thoái hóa)** gây mất trọn điểm (2/3 lần chạy baseline). Mô hình ở nhiệt độ 0 lặp lại một lệnh hoặc một mẫu lệnh đến khi chạm `recursion_limit`. Skill khó phòng ngừa nhóm này, vì đây là hành vi giải mã của mô hình chứ không phải thiếu tri thức.

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa (`src/lab/subagents.py`): `explorer` (chỉ đọc README, docstring, dữ liệu và báo cáo sự thật, quy ước, bẫy dữ liệu, nguyên nhân gốc), `implementer` (thực hiện thay đổi đã đặc tả đầy đủ, sửa nguyên nhân gốc, chạy test, chỉ báo tệp thật sự đổi), `reviewer` (kiểm tra độc lập, không sửa, trả về danh sách PASS/FAIL theo từng quy tắc). Mỗi `description` viết như chỉ dẫn hành động: dùng khi nào (trước khi sửa, khi thực hiện, sau khi xong) và phải đưa gì vào lời giao việc. Vai trò được tách để có một bước kiểm tra độc lập, nhắm vào nhóm lỗi B và F.
- `subagent_calls` (luồng chính):
  - `code-learn`: 4 lần (2 lần `explorer` để tìm hiểu, 1 lần giao sửa lỗi, 1 lần `reviewer`). Điểm 7/10.
  - `data-learn`: 2 lần (1 lần giao phân tích và tạo `answer.json`, 1 lần review độc lập). Điểm 5/8.
  - `logs-learn`: **0 lần**. Tác tử chính dành phần lớn ngân sách bước để lục hệ thống tệp tìm quy ước (`ls -la /usr/share/doc | grep -i acme`, đọc `site-packages`), rồi chạm `recursion_limit` trước khi kịp giao việc. Điểm 0/9. Ở lần chạy đầu của `code-learn` (bị nhiễu pycache), `subagent_calls` cũng bằng 0 vì tác tử mắc kẹt dịch ngược `.pyc`. Như vậy SUBAGENTS_NOTE chỉ khuyến khích: khi tác tử chính "lạc đường" từ sớm, nó không giao việc.
- Thông tin khi giao việc: lời giao việc khá đầy đủ về mặt kỹ thuật. Ví dụ ở `code-learn`, lời giao cho implementer nêu đường dẫn tương đối, lệnh chạy test, danh sách test đang lỗi. Lời giao cho reviewer có mục "ORIGINAL TASK RULES" và nhấn mạnh docstring là đặc tả. Tác tử chính cũng kiểm tra lại kết quả trước khi dùng: đọc lại `pricing.py`, `export.py`, `report.py`, chạy pytest sau báo cáo của implementer, và đọc `answer.json` sau báo cáo ở `data-learn`. **Thiếu** là các quy ước Acme: tác tử chính không biết, nên subagent cũng không thể biết. Vì vậy subagents đạt 7/7 check kỹ thuật ở `code-learn` nhưng 0/3 check `rule_`, và 5/5 kỹ thuật nhưng 0/3 `rule_` ở `data-learn`. Lưu ý: `trace.md` chỉ có luồng chính, việc bên trong subagent không hiện ra.
- Token và thời gian (tác vụ học): subagents trung bình khoảng 439 nghìn token mỗi lần chạy so với 209 nghìn của baseline (gấp khoảng 2,1 lần). `code-learn` mất 252 giây so với 62 giây.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Số lần chạy curator: 1 (đầu vào là kết quả baseline sau cách ly). Không xóa skill nào, không chạy lại. Curator sinh 3 skill hợp lệ, mỗi skill 6 dòng thân. Skill không nhắc id tác vụ, tên tệp dữ liệu hay đáp án. Các tên xuất hiện (`tests/test_regressions.py`, `CHANGELOG.md`, `## Unreleased`) là tên do quy ước Acme yêu cầu.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `follow-stated-conventions` | Tổng quát: trích mọi quy ước thành checklist, khớp tên và định dạng từng ký tự, xử lý mọi trường hợp docstring nêu. Có hai ví dụ lấy từ tác vụ học (lower-case và thay `-` bằng `_`; tiền ở dạng số nguyên cent), phát biểu như ví dụ chứ không như quy tắc cứng. | Đúng với `detail`. Không có chỉ dẫn gây hại. Điểm yếu: không nêu nội dung của quy ước nào ngoài hai ví dụ, nên chỉ giúp khi đề hoặc dữ liệu nêu quy ước. | 6 dòng. `description` "Use when a task specifies explicit output conventions ..." đủ rộng. Được đọc ở cả 3 tác vụ (`skills_read = 3`). |
| `regression-and-changelog` | Khá riêng cho họ `code`: chép gần nguyên văn ba quy tắc `rule_type_hints`, `rule_regression_tests`, `rule_changelog`. Đây là quy ước tổ chức (được phép), không phải đáp án. | Đúng với `detail`. Gộp quy tắc type hints vào skill "regression-and-changelog" nên tên chưa phản ánh hết nội dung. | 6 dòng. `description` nêu đúng tình huống (sửa lỗi kèm quy ước test hồi quy và changelog). Được đọc ở cả 3 tác vụ, kể cả `data-learn`/`logs-learn`, nơi nó không liên quan, vì SKILLS_NOTE yêu cầu đọc mọi skill "could apply". |
| `verify-before-finish` | Tổng quát: liệt kê mọi tệp đầu ra, chạy test, đối chiếu quy ước, "Do not stop after a tool call that only shows a fragment of output". | Đúng. Quy tắc 5 nhắm vào vòng lặp `sed -n 'Np'` của baseline `code-learn`, nhưng không đủ để chặn các vòng lặp khác. | 6 dòng. `description` rộng ("produce output files ... a grader will inspect"). Được đọc ở cả 3 tác vụ. |

Kết quả Phần 3.4 (`results/skills-auto-dev/`): `code-learn` 10/10 (baseline 1/10; cả 3 check `rule_` đều đạt, trace cho thấy tác tử tạo `tests/test_regressions.py`, sửa `CHANGELOG.md` dưới `## Unreleased` và kiểm tra annotation bằng `inspect`). `data-learn` 0/8 và `logs-learn` 0/9, cả hai `GraphRecursionError` dù đã đọc đủ 3 skill. `data-learn` lặp lại một lệnh phân tích; `logs-learn` gọi `read_file` với `offset` tăng từng dòng (155, 156, 157, ...) sau khi đã đọc hết tệp. Skill được **đọc** nhưng không ngăn được vòng lặp thoái hóa (nhóm G).

## 7. Kết quả so sánh (Phần 4.3, 4.4)

(điền sau khi chạy tác vụ đánh giá)

## 8. Phân tích

(điền sau khi chạy tác vụ đánh giá)

## 9. Hạn chế và tính hợp lệ

(điền sau khi chạy tác vụ đánh giá)

## 10. Kết luận

(điền sau khi chạy tác vụ đánh giá)

## Phụ lục

- Lệnh đã chạy: xem `results/commands.log`.
