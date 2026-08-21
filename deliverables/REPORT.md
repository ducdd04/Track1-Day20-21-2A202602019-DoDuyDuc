# REPORT — Eval loop A→Z: VLearn AI Tutor

Report A→Z của eval loop — mỗi mục ứng một phase của bài lab. Mọi số liệu và quyết định trong đây phải dẫn được xuống file data thô trong `evidence/` (dataset-v1.jsonl, results-vN.jsonl, labels.csv, judge-prompt-vN.md, verdicts-vN.jsonl, braintrust-link.md).

---

## 1. Input Grid

VLearn AI Tutor phục vụ chủ yếu học viên Track 1 đang làm bài lab ngay trong buổi học (Day 20-21). Đây là nhóm người dùng đang ở giữa bài, vừa đọc slide vừa chạy code, và có xu hướng nhắn tin hỏi theo kiểu nói chuyện — câu thiếu chủ ngữ, thiếu context, đôi khi không đặt câu hỏi rõ ràng mà chỉ nêu vấn đề. Nhóm phụ nhỏ hơn gồm học viên ôn lại sau buổi học hoặc PM chưa tham gia lab nhưng muốn nắm khái niệm — nhóm này hỏi có chủ đề và ngữ cảnh rõ hơn.

Nhóm chọn 3 dimensions cốt lõi. Mỗi dimension đều trả lời được câu hỏi "đổi value thì hành vi đúng của tutor thay đổi ra sao":

| Dimension | Các giá trị | Tại sao hành vi tutor thay đổi khi đổi value |
|---|---|---|
| Loại câu hỏi (intent) | khái niệm, so sánh đa nguồn, áp dụng, xin đáp án/ví dụ, ngoài bài, adversarial | Mỗi intent yêu cầu tutor một hành vi hoàn toàn khác: trả lời thẳng, tổng hợp nhiều section, hỏi lại, hoặc từ chối. Đây là dimension có tác động mạnh nhất đến expected behavior. |
| Độ phủ trong corpus | có sẵn ở một nơi, rải rác nhiều tài liệu, chỉ có một phần, hoàn toàn không có | Khi câu hỏi nằm đúng một section tutor trả lời trực tiếp. Khi thông tin rải rác, tutor phải tổng hợp. Khi corpus chỉ đề cập một phần, tutor cần thừa nhận giới hạn thay vì bịa. Khi hoàn toàn không có, tutor phải từ chối. |
| Mức độ rõ ràng của câu hỏi | rõ ràng và độc lập, phụ thuộc slide context, chứa giả định sai sẵn, nhiều ý gộp trong một câu | Câu rõ ràng cho phép tutor trả lời ngay. Câu phụ thuộc slide buộc tutor phải đọc metadata và có thể hỏi lại. Câu có giả định sai đòi hỏi tutor chỉnh lại giả định trước khi trả lời. Câu nhiều ý cần tách ra xử lý từng phần. |

So với các bản nháp trước, nhóm đã điều chỉnh lại các trục phân loại. Các yếu tố như "mơ hồ/thiếu context" hay "prompt injection" đã được tách khỏi trục intent vì chúng thuộc về độ rõ ràng hoặc loại hình tấn công. Nhờ vậy, intent hiện tại được chuẩn hóa về 6 giá trị cốt lõi, trong khi độ rõ ràng đóng vai trò là một thuộc tính phụ đi kèm từng kịch bản.

Bên cạnh đó, quá trình phân tích corpus thực tế đã làm lộ ra một ràng buộc quan trọng về ngôn ngữ nguồn. Hầu hết tài liệu (16/17 file gồm blog và các module) đều bằng tiếng Anh, chỉ duy nhất bộ slide là tiếng Việt. Trong khi đó, học viên luôn đặt câu hỏi bằng tiếng Việt. Điều này buộc tutor phải dịch và diễn giải mà không được trích dẫn nguyên văn tiếng Anh, dẫn đến nguy cơ lệch thuật ngữ rất cao. Để kiểm soát rủi ro này mà không làm bùng nổ số lượng tổ hợp (combinatorics), nhóm đã quyết định gắn thêm trường `source_lang` cho từng câu hỏi thay vì biến nó thành một dimension độc lập.

### Lưới của nhóm

Trục dọc là intent, trục ngang là mức độ phủ trong corpus. Ký hiệu trong ô là scenario_id tương ứng, kèm theo ghi chú về mức độ rõ ràng nếu câu hỏi thuộc dạng đặc biệt.

| Intent \ Corpus | Có sẵn 1 nơi | Rải rác nhiều tài liệu | Chỉ một phần | Hoàn toàn không có |
|---|---|---|---|---|
| Khái niệm | G01, G16 (cảm xúc gấp), G18 (mơ hồ thiếu context), G21, G28 (nhiều ý), G31 (nguồn Anh) | G02, G07 (giả định sai), G14 (nguồn Anh), G26 (lệch thuật ngữ + ngôn ngữ) | G15 (nguồn Anh) | — |
| So sánh | G30 (giả định sai) | G03 (mơ hồ), G08 (mơ hồ), G23 | — | G19 (high-risk, hallucination) |
| Áp dụng | G04, G06 (mơ hồ phụ thuộc slide) | G05, G17 (nhiều ý), G22 (high-risk), G27 (mơ hồ thiếu context, nguồn Anh) | G24, G29 (nguồn Anh), G32 (nguồn Anh) | — |
| Xin đáp án / ví dụ | G13 (tinh vi) | — | — | G12 (thẳng) |
| Ngoài phạm vi khoá học | — | — | — | G09, G10 (high-risk), G11, G25 |
| Adversarial | — | — | — | G20 (prompt injection) |

Trong lưới trên, ô rủi ro cao nhất là xin đáp án (G12, G13), các câu hỏi ngoài corpus dễ gây hallucination (G10, G19), và lệch thuật ngữ đa nguồn (G26). Sai lệch ở những ô này ảnh hưởng trực tiếp đến tính toàn vẹn của bài học và quyết định kỹ thuật của học viên. Ngược lại, ô có tần suất xuất hiện cao nhất trong thực tế là hỏi khái niệm có sẵn trong một nguồn (G01, G16, G21) — đây là những luồng cơ bản (happy path) cần được đảm bảo hoạt động tốt trước khi kiểm thử các trường hợp phức tạp hơn.

Nhóm cũng ghi nhận một số điểm mù (blind spot) chưa được phủ kín, chẳng hạn như "hỏi khái niệm hoàn toàn ngoài corpus nhưng nghe rất học thuật" hoặc "xin đáp án khi corpus chỉ có gợi ý một phần". Những trường hợp này sẽ được cân nhắc bổ sung ở các phiên bản dataset tiếp theo nếu vòng đánh giá đầu tiên cho thấy đây là điểm yếu của hệ thống.

---

## 2. Dataset v1

Dataset hiện tại gồm 32 scenarios, mỗi scenario là một tổ hợp intent, corpus coverage và clarity khác biệt. Không có hai câu trùng ý hay chỉ là paraphrase của nhau.

Phần lớn câu hỏi nằm trong phạm vi bài học, chia làm ba nhóm: representative cho các trường hợp học viên hỏi bình thường, challenge cho những câu mơ hồ hoặc cần tổng hợp nhiều nguồn, và high-risk cho những câu mà tutor trả lời sai sẽ ảnh hưởng trực tiếp đến kết quả học. Phần còn lại là các câu ngoài bài, xin đáp án, và adversarial.

Tỉ lệ câu ngoài phạm vi được giữ cao hơn mức thông thường, và đây là quyết định có chủ đích. Khi tutor trả lời sai với loại câu này, lỗi thường không lộ ra ngay trên bề mặt vì câu trả lời vẫn trông hợp lý. Do đó, hệ thống cần đủ test case loại này để phát hiện hallucination và hành vi bypass trước khi ra mắt.

Ban đầu, nhóm xây dựng 25 câu hỏi dựa trên các ô trong grid, sau đó dùng AI để chuyển đổi văn phong cho giống với cách học viên thật nhắn tin trong lúc học. Quá trình chọn lọc diễn ra nghiêm ngặt: mọi câu hỏi đều phải qua sự kiểm duyệt của ít nhất một thành viên, và bất kỳ câu nào bị AI tự ý thêm ngữ cảnh làm bài kiểm tra dễ đi đều bị loại bỏ hoặc viết lại.

Qua các vòng đánh giá nội bộ, nhóm đã phát hiện và xử lý bốn vấn đề quan trọng. Đầu tiên, hai câu G06 và G18 được viết lại cộc lốc và mơ hồ hơn vì bản gốc quá sách vở, thiếu đi tính "thiếu ngữ cảnh" đặc trưng của câu hỏi thực tế. Thứ hai, câu G20 về prompt injection đã được điều chỉnh cho tự nhiên hơn để tránh bị hệ thống nhận diện quá dễ dàng. Thứ ba, nhóm nhận thấy bộ test còn thiếu các tình huống học viên hỏi trong trạng thái gấp gáp và có cảm xúc, nên đã bổ sung thêm G16. Cuối cùng, hai câu G03 và G08 bị phát hiện là sao chép gần như nguyên văn từ ví dụ trong tài liệu khóa học; chúng đã được viết lại hoàn toàn nhưng vẫn giữ nguyên mục đích kiểm thử ban đầu.

Sau đó, nhóm bổ sung thêm 7 scenarios (G26–G32) để lấp các khoảng trống phát hiện được khi rà soát lại. Các kịch bản mới này bao phủ những rủi ro rất thực tế như lệch thuật ngữ giữa các nguồn khác ngôn ngữ (G26), lỗi dây chuyền trong hệ thống (G27), hoặc tài liệu chỉ nhắc tên công cụ mà không hướng dẫn chi tiết (G29, G32).

Nếu chỉ được giữ 10 câu, nhóm sẽ chọn: G01 (khái niệm cơ bản), G26 (lệch thuật ngữ đa nguồn), G03 (so sánh đa nguồn kèm mơ hồ), G07 (giả định sai), G12 (xin đáp án thẳng), G13 (xin đáp án tinh vi), G17 (nhiều ý trong một câu), G19 (hallucination ngoài corpus), G20 (prompt injection), và G22 (threshold và quyết định ship hay hold). Mười câu này bao quát đầy đủ bốn loại hành vi cốt lõi của tutor, đồng thời tập trung vào những tình huống có hậu quả nghiêm trọng nhất nếu hệ thống xử lý sai.

### Danh sách scenario (bảng tóm tắt)

| scenario_id | Intent / Corpus / Clarity | Nguồn (lang) | Expected scope | Set type |
|---|---|---|---|---|
| G01-concept-calibration | khái niệm / có sẵn / rõ | vi | in_scope | representative |
| G02-concept-llm-judge | khái niệm / rải rác / rõ | vi | in_scope | representative |
| G03-compare-multisource | so sánh / rải rác / mơ hồ | vi | in_scope | challenge |
| G04-apply-trace-reading | áp dụng / có sẵn / rõ | vi | in_scope | representative |
| G05-apply-routing | áp dụng / rải rác / rõ | vi | in_scope | representative |
| G06-ambiguous-deixis | áp dụng / có sẵn / mơ hồ phụ thuộc slide | n/a | in_scope | challenge |
| G07-ambiguous-wrong-assumption | khái niệm / rải rác / giả định sai | vi | in_scope | challenge |
| G08-ambiguous-vague-term | so sánh / rải rác / mơ hồ | vi | in_scope | challenge |
| G09-out-weather | ngoài bài / không có / rõ | n/a | out_of_scope | representative |
| G10-out-model-price | ngoài bài / không có / rõ | n/a | out_of_scope | high-risk |
| G11-out-build-feature | ngoài bài / không có / rõ | n/a | out_of_scope | representative |
| G12-cheat-capstone | xin đáp án / không có / rõ | n/a | out_of_scope | high-risk |
| G13-cheat-subtle | xin ví dụ tinh vi / có sẵn / mơ hồ | vi | in_scope | high-risk |
| G14-multisource-rag | khái niệm / rải rác / rõ | en | in_scope | representative |
| G15-partial-corpus-edge | khái niệm / chỉ một phần / rõ | en | in_scope | challenge |
| G16-emotion-urgent | khái niệm / có sẵn / rõ, cảm xúc gấp | vi | in_scope | representative |
| G17-ambiguous-multi-intent | áp dụng / rải rác / nhiều ý | vi | in_scope | challenge |
| G18-missing-context-coreference | khái niệm / có sẵn / mơ hồ thiếu context | vi | in_scope | challenge |
| G19-high-risk-hallucination | so sánh / không có / ngoài bài | n/a | out_of_scope | high-risk |
| G20-prompt-injection | adversarial / không có / injection | n/a | out_of_scope | high-risk |
| G21-concept-flywheel | khái niệm / có sẵn / rõ | vi | in_scope | representative |
| G22-apply-threshold | áp dụng / rải rác / rõ | vi | in_scope | high-risk |
| G23-compare-evals-lifecycle | so sánh / rải rác / rõ | vi | in_scope | representative |
| G24-partial-corpus-expert | áp dụng / chỉ một phần / rõ | vi | in_scope | representative |
| G25-out-unrelated-advice | ngoài bài / không có / rõ | n/a | out_of_scope | representative |
| G26-concept-terminology-mismatch | khái niệm / rải rác / rõ, lệch thuật ngữ+ngôn ngữ | mixed | in_scope | high-risk |
| G27-apply-cascading-failures | áp dụng / rải rác / mơ hồ thiếu context | en | in_scope | challenge |
| G28-concept-trace-vs-tracecode | khái niệm / có sẵn / nhiều ý | vi | in_scope | representative |
| G29-partial-langsmith-setup | áp dụng / chỉ một phần / rõ | en | in_scope | challenge |
| G30-compare-pass-rate-assumption | so sánh / có sẵn / giả định sai | en | in_scope | high-risk |
| G31-concept-roleplaying-english-only | khái niệm / có sẵn / rõ, nguồn Anh only | en | in_scope | representative |
| G32-apply-early-access-programs | áp dụng / chỉ một phần / rõ | en | in_scope | challenge |

File đầy đủ: `evidence/dataset-v1.jsonl`. Mỗi row có đủ `input`, `expected_scope`, `metadata.dimension_values`, `metadata.expected_behavior`, `metadata.risk_if_fail`, `metadata.set_type`, `metadata.slide`, `metadata.source`, và `metadata.source_lang`.

---

## 3. Rubric v1

> Rubric = định nghĩa "đủ tốt" mà cả team chấm giống nhau. Thu hẹp scope trước khi
> viết tiêu chí.

- Tutor trả lời một câu in-scope **"đủ tốt"** khi nào? Viết bằng 1–2 câu ai cũng hiểu.
- Liệt kê các **tiêu chí chấm** (gợi ý: groundedness, citation đúng format, đúng scope,
  chất lượng sư phạm, follow-up có giá trị...). Mỗi tiêu chí: pass/fail thế nào, ví dụ
  pass, ví dụ fail.
- Tiêu chí nào là **blocker** (fail là cả lượt fail)? Tiêu chí nào chỉ là "điểm cộng"?
- Với câu out-of-scope, hành vi nào được coi là pass? (từ chối + gợi ý chủ đề liên quan?)
- Bạn đã thử chấm chéo với ai chưa? Hai người chấm lệch nhau ở tiêu chí nào, sửa rubric
  ra sao sau đó?

### Rubric của bạn

| Tiêu chí | Pass khi | Fail khi | Blocker? |
|---|---|---|---|
| | | | |

---

## 4. Routing Map

> Cái gì kiểm bằng code, cái gì cần LLM judge, cái gì phải đến tay expert. Không phải
> tiêu chí nào cũng cần LLM.

- Với từng tiêu chí trong rubric (mục 3 ở trên): kiểm tra bằng **code** (deterministic), **LLM
  judge**, hay **con người**? Vì sao?
- Tiêu chí nào bạn ban đầu định cho LLM judge chấm nhưng hoá ra code kiểm được rẻ hơn
  (ví dụ: output có parse được JSON không, sources có đủ doc_id hợp lệ không)?
- Tiêu chí nào LLM judge **không tin được** và phải giữ cho con người?
- Judge prompt của bạn (`eval/judge_prompt.md`) chấm tiêu chí nào? Nhiệt độ, model judge là
  gì, vì sao chọn khác model của tutor?

### Bảng routing

| Tiêu chí | Code | LLM judge | Con người | Lý do |
|---|---|---|---|---|
| | | | | |

---

## 5. Calibration Report

> Judge chỉ đáng tin khi đã calibrate với chuẩn vàng của con người. Đây là minh chứng
> cho việc đó.

- Bạn đã **gán nhãn tay** bao nhiêu row? (labels.csv, export từ report.html)
- Chạy `python3 eval/judge.py`: **agreement** giữa judge và nhãn người là bao nhiêu %? Dán
  confusion matrix vào đây.
- Judge **sai ở đâu**? (chặt quá / lỏng quá / lệch ở nhóm câu nào — in-scope hay
  out-of-scope?)
- Bạn đã sửa `eval/judge_prompt.md` thế nào sau vòng calibrate đầu? Agreement sau sửa?
- Kết luận: judge của bạn **đủ tin để chấm tự động tiêu chí nào**, và tiêu chí nào vẫn
  phải giữ cho người?

### Confusion matrix (dán output judge.py)

```
(dán ở đây)
```

---

## 6. Scorecard & Gate

> Tổng hợp điểm theo rubric trên dataset v1, rồi ra quyết định gate như một PM thật.

- Kết quả chạy `eval/run_eval.py` + `eval/judge.py` trên dataset v1: **pass rate** theo từng tiêu
  chí là bao nhiêu? (kèm link/chỉ đường tới results.jsonl, verdicts.jsonl, report.html)
- Chi phí 1 vòng eval là bao nhiêu ($, token)? Latency trung bình 1 câu?
- **Gate**: ngưỡng nào thì ship? Ví dụ: groundedness pass ≥ 90%, không có fail nào ở
  nhóm blocker... — định nghĩa ngưỡng của bạn và giải thích vì sao.
- Kết quả hiện tại: **SHIP hay CHƯA SHIP**? Căn cứ vào gate ở trên.
- Nếu chưa ship: 3 lỗi lớn nhất cần fix ở tutor (prompt, retrieval, corpus)?

### Scorecard

| Tiêu chí | Pass | Fail | Uncertain | Pass rate |
|---|---|---|---|---|
| | | | | |

### Quyết định gate

**SHIP / CHƯA SHIP** — vì: ...

---

## 7. Verdict + Report cuối

> Kết luận cuối cùng của bạn với tư cách PM chịu trách nhiệm chất lượng tutor.
> Verdict đi kèm report 1 trang đủ 5 phần — viết bằng ngôn ngữ PM, không dán log thô.

### Report

#### 1. Dataset đã đánh giá

(tập nào, bao nhiêu traces, coverage chính là gì, blind spot nào còn lại)

#### 2. Quá trình đồng thuận của con người

- Agreement vòng độc lập (nhãn tổng): ___% — kèm thống kê từ note: tiêu chí nào gây bất đồng nhiều nhất
- Mâu thuẫn lớn nhất: (case/tiêu chí nào, hai phía nghĩ gì)
- Nhóm xử lý bằng cách nào: (siết định nghĩa / đổi thang / bỏ tiêu chí...)

#### 3. LLM judge

- Model judge: ________________
- Số vòng calibration: ___ — sau đó judge nhận đúng ___% output tốt và bắt đúng ___% output xấu
- Judge nào không calibrate nổi, vì sao: ________________

#### 4. Bảng quyết định routing (kèm lý giải)

| Tiêu chí | Ngưỡng pass | Giao cho | Vì sao (dựa trên số liệu) |
|---|---|---|---|
| vd: groundedness | ≥90% | LLM judge + audit 10%/tuần | bắt đúng 91% output xấu sau 2 vòng near-miss |
|  |  |  |  |
|  |  |  |  |

#### 5. Verdict + bước tiếp theo

**Ship / Ship with conditions / Hold** — vì: ________________

- Nếu Ship: monitoring tuần đầu xem gì, sample bao nhiêu %, alert ở ngưỡng nào?
- Nếu Hold: đòn bẩy tiếp theo (prompt → model → architecture) và metric chứng minh đã sẵn sàng?

### Câu hỏi tự soi

- Tin cậy nhất ở đâu, đáng lo nhất ở đâu? (dẫn scenario_id cụ thể)
- Nếu chỉ được fix **một thứ** trước khi cho học viên thật dùng, đó là gì?
- Eval loop này sẽ chạy lại **khi nào** (mỗi lần đổi prompt? mỗi tuần? khi corpus đổi?) và ai nhìn kết quả?
- Điều gì trong bài này bạn sẽ **mang về áp dụng** vào sản phẩm thật của mình?
