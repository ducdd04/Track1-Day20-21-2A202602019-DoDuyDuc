# REPORT — Eval loop A→Z: VLearn AI Tutor

Report A→Z của eval loop — mỗi mục ứng một phase của bài lab. Mọi số liệu và quyết
định trong đây phải dẫn được xuống file data thô trong `evidence/` (dataset-v1.jsonl,
results-vN.jsonl, labels.csv, judge-prompt-vN.md, verdicts-vN.jsonl, braintrust-link.md).


---

## 1. Input Grid

VLearn AI Tutor phục vụ chính **học viên đang làm bài lab** (Track 1, Day 20–21). Họ ở giữa bài, đang vừa đọc slide vừa chạy code, hỏi theo kiểu nhắn tin — không hỏi chuẩn.

Nhóm phụ nhỏ hơn: học viên ôn lại sau buổi học, hoặc PM/PO chưa làm lab nhưng muốn hiểu khái niệm. Nhóm này hỏi có chủ đề rõ hơn.

**Dimensions chọn và lý do:**

| Dimension | Values | Vì sao behavior đổi? |
|---|---|---|
| **Loại câu hỏi (Intent)** | khái niệm · so sánh đa nguồn · áp dụng · mơ hồ/thiếu context · xin đáp án · ngoài bài | Quyết định tutor phải trả lời / tổng hợp / hỏi lại / từ chối — hành vi đúng khác nhau hoàn toàn |
| **Độ phủ trong corpus** | có sẵn 1 nơi · rải rác nhiều tài liệu · chỉ có một phần · không có | Tutor trả lời trực tiếp / phải tổng hợp / phải thừa nhận giới hạn / phải từ chối |
| **Mức độ rõ ràng của câu hỏi** | rõ ràng độc lập · phụ thuộc slide context · giả định sai sẵn · nhiều ý cùng lúc | Tutor trả lời ngay / phải đọc slide metadata / phải chỉnh lại giả định / phải tách câu hỏi |

### Lưới của nhóm

| Intent × Corpus | Có sẵn | Rải rác | Chỉ một phần | Không có |
|---|---|---|---|---|
| **Khái niệm** | G01, G21, G16 | G02, G14 | G15 | G19 ⚠️ high-risk |
| **So sánh** | — | G03, G08, G23 | — | — |
| **Áp dụng** | G04, G05, G22 ⚠️ | — | G24 | — |
| **Mơ hồ / thiếu context** | G06, G18 | G07, G17 | — | — |
| **Xin đáp án (adversarial)** | G13 ⚠️ | — | — | G12 ⚠️ |
| **Ngoài bài** | — | — | — | G09, G10, G11, G25 |
| **Prompt injection** | — | — | — | G20 ⚠️ |

**Ô rủi ro cao nhất:** Xin đáp án (G12, G13) và hallucination ngoài corpus (G10, G19) — sai ở đây ảnh hưởng trực tiếp đến integrity bài học và quyết định của học viên. Ô tần suất cao nhất trong thực tế: câu hỏi khái niệm có sẵn (G01, G02, G21).

---

## 2. Dataset v1

**25 rows.** Mỗi row đại diện một combination intent × corpus coverage × clarity khác biệt. Không có 2 rows trùng ý.

**Phân bổ:**
- In-scope: 18 rows (72%) — học viên hỏi đúng bài, tutor phải trả lời từ corpus
- Out-of-scope: 7 rows (28%) — ngoài bài, xin đáp án, adversarial; tutor phải từ chối

Trong in-scope: representative 12 · challenge 7 (mơ hồ, đa nguồn, giả định sai, nhiều ý) · high-risk 6.

**Tỉ lệ out-of-scope 28% cao hơn thông thường là cố ý** — tutor trả lời sai khi hỏi ngoài bài không lộ ngay, cần đủ test case để phát hiện hallucination và bypass.

**Nguồn câu hỏi:** Toàn bộ 25 câu do nhóm đặt theo từng combination trong grid rồi AI (Claude) paraphrase thành giọng học viên thật. Mỗi câu qua lọc Keep/Rewrite/Reject — câu nào AI tự thêm context làm bài dễ hơn đều bị Rewrite hoặc Reject.

**Review nội bộ phát hiện:**
- 2 câu ban đầu quá sách vở, thiếu tính thiếu-context — đã viết lại thêm cộc hơn (G06, G18)
- 1 câu "prompt injection" ban đầu quá dễ nhận diện — đã làm mượt hơn để test thật (G20)
- Thiếu case học viên gấp/cảm xúc — bổ sung G16

**10 câu ưu tiên nhất nếu chỉ giữ 10:**
G01 (calibration cơ bản), G02 (LLM judge cơ bản), G03 (so sánh đa nguồn + mơ hồ), G07 (giả định sai), G12 (xin đáp án thẳng), G13 (xin đáp án tinh vi), G17 (nhiều ý cùng lúc), G19 (hallucination high-risk), G20 (prompt injection), G22 (threshold quyết định ship/hold).
Lý do: 10 rows này phủ đủ 4 loại hành vi khác nhau cốt lõi của tutor, và tập trung vào chỗ failure cost cao nhất.

### Danh sách scenario (bảng tóm tắt)

| scenario_id | Intent × Corpus × Clarity | expected scope | set_type |
|---|---|---|---|
| G01-concept-calibration | khái niệm / có sẵn / rõ | in_scope | representative |
| G02-concept-llm-judge | khái niệm / rải rác / rõ | in_scope | representative |
| G03-compare-multisource | so sánh / rải rác / mơ hồ | in_scope | challenge |
| G04-apply-trace-reading | áp dụng / có sẵn / rõ | in_scope | representative |
| G05-apply-routing | áp dụng / rải rác / rõ | in_scope | representative |
| G06-ambiguous-deixis | áp dụng / có sẵn / mơ hồ + slide | in_scope | challenge |
| G07-ambiguous-wrong-assumption | khái niệm / rải rác / giả định sai | in_scope | challenge |
| G08-ambiguous-vague-term | so sánh / rải rác / mơ hồ | in_scope | challenge |
| G09-out-weather | ngoài bài / không có / rõ | out_of_scope | representative |
| G10-out-model-price | ngoài bài / không có / rõ | out_of_scope | high-risk |
| G11-out-build-feature | ngoài bài / không có / rõ | out_of_scope | representative |
| G12-cheat-capstone | xin đáp án / không có / rõ | out_of_scope | high-risk |
| G13-cheat-subtle | xin ví dụ (tinh vi) / có sẵn / mơ hồ | in_scope | high-risk |
| G14-multisource-rag | khái niệm / rải rác / rõ | in_scope | representative |
| G15-partial-corpus-edge | khái niệm / chỉ một phần / rõ | in_scope | challenge |
| G16-emotion-urgent | khái niệm / có sẵn / rõ + cảm xúc | in_scope | representative |
| G17-ambiguous-multi-intent | áp dụng / rải rác / nhiều ý | in_scope | challenge |
| G18-missing-context-coreference | khái niệm / có sẵn / mơ hồ thiếu context | in_scope | challenge |
| G19-high-risk-hallucination | so sánh / không có / ngoài bài | out_of_scope | high-risk |
| G20-prompt-injection | adversarial / không có / injection | out_of_scope | high-risk |
| G21-concept-flywheel | khái niệm / có sẵn / rõ | in_scope | representative |
| G22-apply-threshold | áp dụng / rải rác / rõ | in_scope | high-risk |
| G23-compare-evals-lifecycle | so sánh / rải rác / rõ | in_scope | representative |
| G24-partial-corpus-expert | áp dụng / chỉ một phần / rõ | in_scope | representative |
| G25-out-unrelated-advice | ngoài bài / không có / rõ | out_of_scope | representative |

*File đầy đủ: `evidence/dataset-v1.jsonl` — mỗi row có đủ `input`, `expected_scope`, `metadata.dimension_values`, `metadata.expected_behavior`, `metadata.risk_if_fail`, `metadata.set_type`, `metadata.slide`.*



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
