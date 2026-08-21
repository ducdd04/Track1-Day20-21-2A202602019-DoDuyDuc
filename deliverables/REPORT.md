# REPORT — Eval loop A→Z: VLearn AI Tutor

Report A→Z của eval loop — mỗi mục ứng một phase của bài lab. Mọi số liệu và quyết
định trong đây phải dẫn được xuống file data thô trong `evidence/` (dataset-v1.jsonl,
results-vN.jsonl, labels.csv, judge-prompt-vN.md, verdicts-vN.jsonl, braintrust-link.md).


---

## 1. Input Grid

VLearn AI Tutor phục vụ chủ yếu học viên Track 1 đang làm bài lab ngay trong buổi học (Day 20-21). Đây là nhóm người dùng đang ở giữa bài, vừa đọc slide vừa chạy code, và có xu hướng nhắn tin hỏi theo kiểu nói chuyện -- câu thiếu chủ ngữ, thiếu context, đôi khi không đặt câu hỏi rõ ràng mà chỉ nêu vấn đề. Nhóm phụ nhỏ hơn gồm học viên ôn lại sau buổi học hoặc PM chưa tham gia lab nhưng muốn nắm khái niệm -- nhóm này hỏi có chủ đề và ngữ cảnh rõ hơn.

Nhóm chọn 3 dimensions sau, mỗi dimension đều trả lời được câu hỏi "đổi value thì hành vi đúng của tutor thay đổi ra sao":

| Dimension | Các giá trị | Tại sao hành vi tutor thay đổi khi đổi value |
|---|---|---|
| Loại câu hỏi (intent) | khái niệm, so sánh đa nguồn, áp dụng, mơ hồ/thiếu context, xin đáp án, ngoài bài | Mỗi intent yêu cầu tutor một hành vi hoàn toàn khác: trả lời thẳng, tổng hợp nhiều section, hỏi lại, hoặc từ chối. Đây là dimension có tác động mạnh nhất đến expected behavior. |
| Độ phủ trong corpus | có sẵn ở một nơi, rải rác nhiều tài liệu, chỉ có một phần, hoàn toàn không có | Khi câu hỏi nằm đúng một section tutor trả lời trực tiếp. Khi thông tin rải rác, tutor phải tổng hợp. Khi corpus chỉ đề cập một phần, tutor cần thừa nhận giới hạn thay vì bịa. Khi hoàn toàn không có, tutor phải từ chối. |
| Mức độ rõ ràng của câu hỏi | rõ ràng và độc lập, phụ thuộc slide context, chứa giả định sai sẵn, nhiều ý gộp trong một câu | Câu rõ ràng cho phép tutor trả lời ngay. Câu phụ thuộc slide buộc tutor phải đọc metadata và có thể hỏi lại. Câu có giả định sai đòi hỏi tutor chỉnh lại giả định trước khi trả lời. Câu nhiều ý cần tách ra xử lý từng phần. |

### Lưới của nhóm

Trục dọc là intent, trục ngang là mức độ phủ trong corpus. Ký hiệu trong ô là scenario_id tương ứng.

| Intent / Corpus | Có sẵn 1 nơi | Rải rác nhiều tài liệu | Chỉ một phần | Hoàn toàn không có |
|---|---|---|---|---|
| Khái niệm | G01, G16, G21 | G02, G14 | G15 | G19 (high-risk) |
| So sánh | -- | G03, G08, G23 | -- | -- |
| Áp dụng | G04, G05 | -- | G24 | -- |
| Áp dụng (high-risk) | G22 | -- | -- | -- |
| Mơ hồ / thiếu context | G06, G18 | G07, G17 | -- | -- |
| Xin đáp án (adversarial) | G13 (tinh vi) | -- | -- | G12 (thẳng) |
| Ngoài bài | -- | -- | -- | G09, G10, G11, G25 |
| Prompt injection | -- | -- | -- | G20 |

Ô rủi ro cao nhất là xin đáp án (G12, G13) và câu hỏi ngoài corpus dễ gây hallucination (G10, G19) -- sai ở hai ô này ảnh hưởng trực tiếp đến tính toàn vẹn bài học và quyết định kỹ thuật của học viên. Ô tần suất cao nhất trong thực tế là hỏi khái niệm có sẵn trong một nguồn (G01, G02, G21) -- đây là happy path cần làm tốt trước khi kiểm các ô khó hơn.

---

## 2. Dataset v1

Dataset gồm 25 scenarios, mỗi scenario là một tổ hợp intent, corpus coverage và clarity khác biệt. Không có hai câu trùng ý hay chỉ là paraphrase của nhau.

Phần lớn câu hỏi nằm trong phạm vi bài học, trong đó chia làm ba nhóm: representative cho các trường hợp học viên hỏi bình thường, challenge cho những câu mơ hồ hoặc cần tổng hợp nhiều nguồn, và high-risk cho những câu mà tutor trả lời sai sẽ ảnh hưởng trực tiếp đến kết quả học. Phần còn lại là các câu ngoài bài, xin đáp án, và adversarial.

Tỉ lệ câu ngoài phạm vi được giữ cao hơn mức thông thường, và đây là quyết định có chủ đích. Khi tutor trả lời sai với loại câu này, lỗi thường không lộ ra ngay trên bề mặt vì câu trả lời vẫn trông hợp lý. Cần đủ test case loại này để phát hiện hallucination và hành vi bypass trước khi ship.

Toàn bộ 25 câu do nhóm đặt theo từng ô trong grid, sau đó dùng Claude để paraphrase thành giọng học viên thật nhắn tin trong lúc học. Mỗi câu đều qua vòng quyết định Keep, Rewrite hoặc Reject của ít nhất một thành viên. Câu nào Claude tự thêm context khiến bài dễ hơn đều bị Rewrite hoặc Reject, không giữ nguyên.

Review nội bộ phát hiện ba vấn đề và xử lý trực tiếp. Hai câu G06 và G18 viết quá sách vở, thiếu tính thiếu-context của câu hỏi thật nên được viết lại cộc và mơ hồ hơn. Câu G20 về prompt injection ban đầu quá lộ liễu, dễ bị nhận diện ngay nên được làm mượt để test thật hơn. Ngoài ra nhóm nhận ra thiếu hẳn case học viên hỏi trong trạng thái gấp và có cảm xúc, nên bổ sung thêm G16.

Nếu chỉ giữ 10 câu, nhóm chọn: G01 (calibration cơ bản), G02 (LLM judge cơ bản), G03 (so sánh đa nguồn kèm mơ hồ), G07 (giả định sai), G12 (xin đáp án thẳng), G13 (xin đáp án tinh vi), G17 (nhiều ý trong một câu), G19 (hallucination ngoài corpus), G20 (prompt injection), G22 (threshold và quyết định ship hay hold). Mười câu này phủ đủ bốn loại hành vi cốt lõi của tutor và tập trung vào những chỗ failure cost cao nhất.

### Danh sách scenario (bảng tóm tắt)

| scenario_id | Intent / Corpus / Clarity | Expected scope | Set type |
|---|---|---|---|
| G01-concept-calibration | khái niệm / có sẵn / rõ | in_scope | representative |
| G02-concept-llm-judge | khái niệm / rải rác / rõ | in_scope | representative |
| G03-compare-multisource | so sánh / rải rác / mơ hồ | in_scope | challenge |
| G04-apply-trace-reading | áp dụng / có sẵn / rõ | in_scope | representative |
| G05-apply-routing | áp dụng / rải rác / rõ | in_scope | representative |
| G06-ambiguous-deixis | áp dụng / có sẵn / mơ hồ phụ thuộc slide | in_scope | challenge |
| G07-ambiguous-wrong-assumption | khái niệm / rải rác / giả định sai | in_scope | challenge |
| G08-ambiguous-vague-term | so sánh / rải rác / mơ hồ | in_scope | challenge |
| G09-out-weather | ngoài bài / không có / rõ | out_of_scope | representative |
| G10-out-model-price | ngoài bài / không có / rõ | out_of_scope | high-risk |
| G11-out-build-feature | ngoài bài / không có / rõ | out_of_scope | representative |
| G12-cheat-capstone | xin đáp án / không có / rõ | out_of_scope | high-risk |
| G13-cheat-subtle | xin ví dụ tinh vi / có sẵn / mơ hồ | in_scope | high-risk |
| G14-multisource-rag | khái niệm / rải rác / rõ | in_scope | representative |
| G15-partial-corpus-edge | khái niệm / chỉ một phần / rõ | in_scope | challenge |
| G16-emotion-urgent | khái niệm / có sẵn / rõ, có cảm xúc gấp | in_scope | representative |
| G17-ambiguous-multi-intent | áp dụng / rải rác / nhiều ý | in_scope | challenge |
| G18-missing-context-coreference | khái niệm / có sẵn / mơ hồ thiếu context | in_scope | challenge |
| G19-high-risk-hallucination | so sánh / không có / ngoài bài | out_of_scope | high-risk |
| G20-prompt-injection | adversarial / không có / injection | out_of_scope | high-risk |
| G21-concept-flywheel | khái niệm / có sẵn / rõ | in_scope | representative |
| G22-apply-threshold | áp dụng / rải rác / rõ | in_scope | high-risk |
| G23-compare-evals-lifecycle | so sánh / rải rác / rõ | in_scope | representative |
| G24-partial-corpus-expert | áp dụng / chỉ một phần / rõ | in_scope | representative |
| G25-out-unrelated-advice | ngoài bài / không có / rõ | out_of_scope | representative |

File đầy đủ: evidence/dataset-v1.jsonl. Mỗi row có đủ input, expected_scope, metadata.dimension_values, metadata.expected_behavior, metadata.risk_if_fail, metadata.set_type và metadata.slide.



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
