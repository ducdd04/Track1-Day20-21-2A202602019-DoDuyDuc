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

| Tiêu chí | Pass khi | Fail khi | Blocker |
|---|---|---|---|
| Mức độ bám sát nguồn dữ liệu | Sinh viên trích dẫn đúng nguồn gốc dữ liệu trong tài liệu. | Sinh viên tự suy diễn hoặc bịa thông tin sai lệch so với tài liệu gốc. | Có |
| Kiểm soát phạm vi câu hỏi | Từ chối trả lời các yêu cầu nằm ngoài phạm vi môn học. | Trả lời sai phạm vi hoặc thực hiện các hành vi không được phép như viết mã hộ. | Có |
| Làm rõ câu hỏi | Đặt câu hỏi làm rõ khi học viên đưa ra yêu cầu không đầy đủ ngữ cảnh. | Tự đưa ra giả định để giải thích khi câu hỏi không có đủ thông tin. | Không nhưng trừ điểm |

---

## 4. Routing Map

> Cái gì kiểm bằng code, cái gì cần LLM judge, cái gì phải đến tay expert. Không phải
> tiêu chí nào cũng cần LLM.

### Bảng routing

| Tiêu chí | Code | LLM judge | Con người | Lý do |
|---|---|---|---|---|
| Kiểm tra định dạng | Có | | | Sử dụng mã lập trình để kiểm tra định dạng và nguồn trích dẫn mang lại độ chính xác tuyệt đối với chi phí thấp nhất. |
| Mức độ bám sát nguồn | | Có | | Việc đánh giá mức độ bám sát nguồn dữ liệu cần khả năng xử lý ngôn ngữ tự nhiên dựa trên các quy tắc được định nghĩa trước. |
| Yếu tố cảm xúc | | | Có | Các câu hỏi có yếu tố tâm lý hoặc thiếu ngữ cảnh rõ ràng thường khiến mô hình ngôn ngữ đánh giá sai do tính cứng nhắc. |

---

## 5. Calibration Report

> Judge chỉ đáng tin khi đã calibrate với chuẩn vàng của con người. Đây là minh chứng
> cho việc đó.

Nhóm đã tiến hành dán nhãn thủ công cho 32 mẫu dữ liệu. Độ đồng thuận ban đầu giữa ba thành viên đạt mức 56 phần trăm.

Trong lần chạy đầu tiên với lời nhắc gốc, mức độ đồng thuận giữa mô hình và con người chỉ đạt 50 phần trăm. Mô hình giám khảo đánh giá quá khắt khe, dẫn đến tình trạng từ chối sai nhiều câu trả lời hợp lệ.

Sau khi điều chỉnh lời nhắc và bổ sung các ví dụ đạt yêu cầu, mức độ đồng thuận tăng lên 62 phần trăm.

Mô hình nemotron-30b vẫn có xu hướng đánh giá an toàn quá mức, dẫn đến 10 trường hợp bị từ chối sai. Do đó, nhóm quyết định chỉ sử dụng mô hình này như một công cụ hỗ trợ phân loại ban đầu. Quyết định đối với các trường hợp phức tạp sẽ do chuyên gia thực hiện.

### Confusion matrix

```text
           |      pass      fail uncertain       
      pass |        10         1         1       
      fail |        10        10         0       
 uncertain |         0         0         0       
```

---

## 6. Scorecard & Gate

> Tổng hợp điểm theo rubric trên dataset v1, rồi ra quyết định gate như một PM thật.

Quá trình kiểm thử cục bộ không phát sinh chi phí giao tiếp qua giao diện lập trình ứng dụng. Độ trễ trung bình cho mỗi truy vấn dao động từ 2 đến 3 giây.

Ngưỡng phê duyệt được xác định như sau. Kiểm tra tính hợp lệ của cấu trúc dữ liệu yêu cầu đạt 100 phần trăm. Kiểm tra tính tồn tại của nguồn trích dẫn yêu cầu đạt 100 phần trăm. Độ chính xác của đoạn trích dẫn nguyên văn yêu cầu đạt tối thiểu 95 phần trăm. Mức độ bám sát nguồn tài liệu yêu cầu đạt tối thiểu 80 phần trăm.

### Scorecard

| Tiêu chí | Pass | Fail | Uncertain | Pass rate |
|---|---|---|---|---|
| Cấu trúc dữ liệu | 32 | 0 | 0 | 100 phần trăm |
| Nguồn trích dẫn | 32 | 0 | 0 | 100 phần trăm |
| Trích dẫn nguyên văn | 26 | 6 | 0 | 81.25 phần trăm |
| Bám sát nguồn tài liệu | 12 | 20 | 0 | 37.5 phần trăm |

### Quyết định gate

Chưa triển khai vì hai nguyên nhân chính. Tỉ lệ trích dẫn nguyên văn chỉ đạt 81.25 phần trăm, cho thấy hệ thống truy xuất và tạo sinh đang gặp lỗi trong việc trích xuất văn bản. Điểm bám sát nguồn tài liệu do mô hình giám khảo chấm ở mức rất thấp, ngay cả khi đối chiếu với nhãn do con người gán thì kết quả vẫn chỉ đạt 62 phần trăm. Hệ thống chưa đạt độ ổn định cần thiết.

Các lỗi cần ưu tiên khắc phục bao gồm lỗi truy xuất văn bản khiến câu trích dẫn bị sai lệch, hiện tượng ảo giác của mô hình gia sư khi thiếu ngữ cảnh và tính khắt khe quá mức của mô hình giám khảo nemotron-30b.

---

## 7. Verdict + Report cuối

> Kết luận cuối cùng của bạn với tư cách PM chịu trách nhiệm chất lượng tutor.
> Verdict đi kèm report 1 trang đủ 5 phần.

### Report

#### 1. Dataset đã đánh giá
Tập dữ liệu thử nghiệm bao gồm 32 mẫu. Tập dữ liệu này bao phủ 6 nhóm ý định chính của người học với 4 mức độ phân bổ tài liệu khác nhau. Hệ thống chưa được đánh giá toàn diện với các kỹ thuật tiêm nhiễm lời nhắc phức tạp hoặc yêu cầu dịch thuật chéo ngôn ngữ.

#### 2. Quá trình đồng thuận của con người
Độ đồng thuận độc lập đạt 56 phần trăm. Bất đồng lớn nhất tập trung ở các câu hỏi thiếu ngữ cảnh, nơi một nhóm muốn mô hình đưa ra giả định để hỗ trợ người dùng, nhóm còn lại yêu cầu mô hình phải xác minh lại thông tin. Nhóm đã thống nhất quy tắc không cho phép mô hình tự suy diễn ngữ cảnh và cập nhật lại bộ nhãn chuẩn.

#### 3. LLM judge
Mô hình giám khảo được sử dụng là nemotron-30b. Sau hai vòng tinh chỉnh lời nhắc, độ đồng thuận với con người đạt 62 phần trăm. Mô hình này không thể hiệu chuẩn để đạt ngưỡng trên 85 phần trăm do xu hướng tránh rủi ro quá mức, dẫn đến tỉ lệ từ chối sai rất cao.

#### 4. Bảng quyết định routing

| Tiêu chí | Ngưỡng pass | Giao cho | Vì sao |
|---|---|---|---|
| Cấu trúc dữ liệu | 100 phần trăm | Kiểm tra tự động | Mã lập trình kiểm tra tính hợp lệ mang lại kết quả tuyệt đối với tốc độ cao. |
| Trích dẫn nguyên văn | Lớn hơn 95 phần trăm | Kiểm tra tự động | Hệ thống tự động phát hiện được 6 lỗi trích xuất sai mà việc đánh giá thủ công dễ bỏ sót. |
| Bám sát nguồn | Lớn hơn 80 phần trăm | Mô hình hỗ trợ | Mô hình hiện tại từ chối sai khá nhiều nên chỉ đóng vai trò phân loại sơ bộ trước khi con người thẩm định. |

#### 5. Verdict + bước tiếp theo

Quyết định tạm hoãn triển khai do hệ thống chưa đáp ứng tiêu chuẩn chất lượng. Tỉ lệ trích dẫn sai sót cao và hệ thống vẫn gặp khó khăn trong việc xử lý các truy vấn thiếu ngữ cảnh.

Hướng khắc phục tiếp theo bao gồm việc điều chỉnh hệ thống truy xuất để giữ nguyên vẹn nội dung trích dẫn, tối ưu hóa lời nhắc hệ thống nhằm giảm thiểu hiện tượng ảo giác và xem xét thay thế mô hình giám khảo nemotron-30b bằng một mô hình có khả năng suy luận tốt hơn để nâng cao độ đồng thuận.

### Câu hỏi tự soi (Phần cá nhân - Đỗ Duy Đức)
> Phần tự soi này dành cho từng cá nhân tự điền, không phải quyết định chung của nhóm.

- **Tin cậy nhất ở đâu, đáng lo nhất ở đâu?**:
  - *Tin cậy nhất*: Khi chịu trách nhiệm thiết kế ma trận Input Grid ở Phase 1 và phân tích dữ liệu Scorecard theo slice, tôi nhận thấy hệ thống Tutor hoạt động rất đáng tin cậy với làn kiểm tra Code-based (đặc biệt là rule `schema_valid`, `citation_exists`, và hàm `check_followup_count` mới bổ sung đạt 100% pass rate) cũng như các câu hỏi khái niệm cơ bản có sẵn trong corpus (như G01, G04).
  - *Đáng lo nhất*: Khi trực tiếp gán nhãn 32 test cases (`labels-duc.csv`), tôi phát hiện hiện tượng "ảo giác nguồn trích dẫn" (citation mismatch - như G14, G15, G21 trích dẫn sai slide không hỗ trợ câu trả lời) và hiện tượng rò rỉ phạm vi (scope leakage - G11 tự động viết code hộ, G13 cho ví dụ giải hộ bài tập capstone).
- **Nếu chỉ được sửa một thứ trước khi cho học viên thật sử dụng, đó là gì?**:
  - Tối ưu thuật toán retrieval và prompt của Tutor để triệt tiêu hiện tượng bịa đặt trích dẫn. Tutor phải bắt buộc trích dẫn đúng đoạn văn bản thực sự hỗ trợ cho câu trả lời; nếu context mơ hồ hoặc không đủ dữ liệu, Tutor phải phản hồi đặt câu hỏi làm rõ (clarify) thay vì tự suy diễn bừa bãi.
- **Vòng lặp đánh giá này sẽ chạy lại khi nào và ai là người đánh giá kết quả?**:
  - Vòng lặp sẽ chạy lại khi cập nhật kho học liệu corpus, khi tinh chỉnh System Prompt của Tutor, hoặc khi chuyển đổi mô hình LLM nền.
  - Đội ngũ giảng viên/trợ giảng (TAs) kết hợp với kỹ sư AI/PM sẽ đứng ra đánh giá; sử dụng làn Code check tự động (bao gồm `check_followup_count`) và LLM Judge đã hiệu chuẩn trước, sau đó chọn mẫu ngẫu nhiên 15-20% để thẩm định thủ công.
- **Điều gì trong bài thực hành này bạn sẽ mang về áp dụng vào sản phẩm thực tế của mình?**:
  - Phương pháp thiết kế ma trận Input Grid đa chiều (Intent x Corpus Coverage x Clarity) để phủ kín các kịch bản thực tế của người dùng.
  - Việc kết hợp kiểm tra 3 làn (Code-based checks như `check_followup_count`, LLM Judge, và Human labeling) để xây dựng Scorecard & Routing Map tối ưu chi phí và độ chính xác cho hệ thống AI.

---

## 8. AI Support Log (Phần cá nhân - Đỗ Duy Đức)

> Phần này mỗi thành viên tự điền để ghi lại quá trình hỗ trợ của trí tuệ nhân tạo đối với cá nhân mình.

- **AI đã giúp tôi ở đâu?**:
  - Gợi ý các ý tưởng kịch bản câu hỏi để tôi xây dựng ma trận Input Grid ở Phase 1.
  - Gợi ý cấu trúc logic và template để tôi viết hàm `check_followup_count` trong `eval/code_checks.py`.
  - Hỗ trợ tổng hợp dữ liệu thống kê theo slice để phân tích viết báo cáo Scorecard.
- **AI sai, hời hợt hoặc làm mất độ bao phủ ở đâu?**:
  - Khi được yêu cầu gợi ý các kịch bản câu hỏi "mơ hồ" hoặc "xin đáp án tinh vi", AI sinh ra các câu hỏi quá rõ ràng, thiếu đi đặc tính cộc lốc và ngắn gọn thực tế của học viên.
  - Khi tự động chấm điểm, AI Judge bỏ sót các lỗi trích dẫn sai nguồn (citation mismatch) và không phát hiện được lỗi viết code hộ (scope leak).
- **Tôi đã tự sửa hoặc quyết định lại điều gì?**:
  - Tự tay tinh chỉnh lại toàn bộ 32 kịch bản trên Input Grid để đảm bảo tính thực tế và đa dạng của các ô rủi ro (high-risk, challenge).
  - Tự tay thực hiện gán nhãn độc lập 32 test cases (`labels-duc.csv`) hoàn toàn bằng đánh giá thủ công của con người và bảo vệ quan điểm trong buổi họp đồng thuận nhãn vàng.
  - Tự tay viết và kiểm thử hàm `check_followup_count` trong `eval/code_checks.py` để đảm bảo số lượng câu hỏi gợi ý luôn đạt từ 1 đến 3 câu.
