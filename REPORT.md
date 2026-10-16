# REPORT — Lab Day 12

Chạy trên máy cá nhân Windows, CPU, bằng `.venv/Scripts/python.exe run_all.py --student-id 2A202602687 --seeds 3`, không dùng `--quick`.
Giữ nguyên cấu hình huấn luyện, số vòng DAgger và ngưỡng cổng của bộ khung; dùng một luồng BLAS/OMP để hạn chế chi phí CPU.

| | |
|---|---|
| Họ và tên | Nguyễn Khánh Đô |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602687` |
| GitHub | Diohus |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 2205 / 5011, 5012, 5013 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Số liệu từ `evidence/closedloop.json`, `openloop.json`, `gate.json`; mô hình học trình bày min–max qua 3 seed.
L2 từng seed PercMLP+ được tính lại từ các trọng số đã lưu, không huấn luyện thêm, trong `evidence/percfix_l2.json`; trung bình 0,831669 m khớp `openloop.json`.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,253633 | 20,7423 | BLOCK |
| IDM | 1,541805 | 0,0347 | (baseline) |
| EgoMLP | 0,761000–0,764408 | 20,6729–23,4131 | BLOCK |
| PercMLP | 0,617739–0,621094 | 0,3469–1,5262 | BLOCK |
| PercMLP+ (sau khi sửa) | 0,813446–0,866636 | 0,1734–0,2081 | RELEASE (7/7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

`evidence/data_audit.json` ghi 992.679 dòng, 9.993 quỹ đạo ban đầu; loại 383 quỹ đạo trùng còn 9.610 quỹ đạo.
Tương quan `Acc_LV` với đạo hàm vận tốc chỉ 0,04535, trong khi cột gia tốc xe theo sau đạt 0,92909; vì vậy không dùng `Acc_LV` để tìm ca phanh.
Thay vào đó, làm mượt `Speed_LV` rồi lấy đạo hàm theo thời gian để có `a_lead`, giảm ảnh hưởng của nhiễu vận tốc.
Các bản sao có ID khác nhau có thể lọt vào cả train và test; cần loại trùng trước khi chia để tránh rò rỉ.
Chia theo quỹ đạo giữ các khung và cửa sổ chồng lấn của cùng chuyến đi trong một tập, tránh việc mô hình được chấm trên chuyển động đã thấy.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Theo `evidence/replay.json`, chạy lại cùng mã giống hệt; float32 khác bit ở 600/600 đoạn nhưng 0 đoạn vượt dung sai, sai khác cực đại chỉ 0,0001222 m/s².
Dùng `atol=rtol=0,001` với ngưỡng `atol + rtol*|golden|` phù hợp kiểm tra refactor số học trong bài; NaN đệm ở cùng vị trí được coi là bằng nhau.
AEB làm lệch 8/600 đoạn; kiểm tra đoạn 82 trong `evidence/aeb_inspection.txt` thấy các khung 75–77 có TTC lần lượt 1,475106; 1,462186; 1,435671 s.
Các TTC này dưới ngưỡng 1,5 s, nên lệnh phanh khoảng −4,38 đến −4,35 được đổi thành −6 m/s²; ngoài điều kiện kích hoạt hoặc khi đã phanh mạnh hơn, AEB không đổi lệnh.
Phát thời gian thật có 585 khung trễ và 579 khung rớt trên 59.924 khung, làm lệch 365 đoạn; phát nhanh nhất có 0 khung trễ/rớt.
Đây là chi phí xử lý được mô phỏng theo timestamp: khi xử lý vượt hạn hoặc còn bận, hệ thống giữ lệnh cũ; phát nhanh nhất không áp đặt deadline nên che lỗi này.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

`evidence/scenarios.json` dùng cruise 10 m/s, khoảng cách 2,760–59,312 m, vận tốc tiến gần 0–10 m/s; lưới 60×45 có 2.700 ô.
Biên bất khả thi là `gap < dv*t_react + dv²/(2*9)` với `dv>0`; tỷ lệ bất khả thi tăng từ 0,963% lên 5,148% khi độ trễ tăng 0 lên 0,6 s.
Vùng va chạm IDM cũng tăng từ 0,963% lên 5,148%, tức 4,185 điểm phần trăm; phần đâm ngoài vùng bất khả thi bằng 0 ở cả hai độ trễ.
Kiểm chứng phanh tối đa có 0 ô lệch ngoài vùng sát biên ở cả hai độ trễ, nên phân biệt được giới hạn vật lý với lỗi điều khiển.
Khai thác được 614 ca phanh (6,389% quỹ đạo); hình `scenarios_idm.png` cho thấy log chủ yếu phủ vận tốc tiến gần thấp, thưa ở vùng tiến gần nhanh và khoảng cách nhỏ.
Mức phanh người lái ở phân vị 99,9% chỉ 3,625 m/s², trong khi giới hạn cơ cấu là 9 m/s²; 5,222% lưới đòi phanh mạnh hơn mức phân vị này, cho thấy thiếu bao phủ tình huống khó.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Lấy trung bình các seed để xếp hạng như `evidence/closedloop.json[ranking]`; tỷ lệ va chạm trong bảng này là trung bình, còn bảng kết quả cuối trình bày dải.

| Bộ điều khiển | L2 trung bình (m) | Hạng L2 | Va chạm trung bình (%) | Hạng vòng kín |
|---|---|---|---|---|
| constant-velocity | 1,253633 | 3 | 20,7423 | 3 |
| IDM | 1,541805 | 4 | 0,0347 | 1 |
| EgoMLP | 0,763246 | 2 | 21,7135 | 4 |
| PercMLP | 0,619818 | 1 | 0,7862 | 2 |

Các cặp đảo hạng là constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP.
IDM không tối ưu bắt chước quỹ đạo người lái nên L2 lớn, nhưng phản ứng trực tiếp theo khoảng cách và tốc độ tiến gần giúp hạn chế va chạm khi tự lái.
EgoMLP dự đoán tốt chuyển động thường gặp từ lịch sử tốc độ, nhưng thiếu khoảng cách và vận tốc xe dẫn; lỗi dự đoán làm trạng thái tự lái lệch tiếp qua từng bước.
Trong `evidence/openloop.json`, lát cắt phanh chỉ chiếm 4,632% cửa sổ, nhưng L2 PercMLP trên lát cắt là 0,856674 m so với 0,619818 m toàn bộ, nên trung bình đẹp không bảo đảm an toàn.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

`evidence/closedloop.json` ghi EgoMLP seed 5011 va chạm 675/2.883 ca (23,4131%), seed 5012 là 596/2.883 (20,6729%), seed 5013 là 607/2.883 (21,0545%).
Dải qua seed rộng 2,7402 điểm phần trăm; đây là biến thiên do huấn luyện dù cùng tập kiểm thử, vượt ngưỡng ổn định 1 điểm của R5.
Wilson 95% của seed tốt nhất là [19,2342%; 22,1897%], seed xấu nhất là [21,9034%; 24,9936%]; hai khoảng có chồng lấn.
Khoảng Wilson thể hiện bất định của tỷ lệ trên số quỹ đạo kiểm thử hữu hạn, không thay thế việc chạy nhiều seed và không chứng minh an toàn ngoài phân phối đó.
Nếu chỉ chọn seed 5012, có thể báo 20,6729% và tưởng tốt hơn constant-velocity 20,7423%; seed 5011 lại cho 23,4131%, làm kết luận đảo chiều.
Vì các seed dùng chung log nên không xem chúng là các tập thử độc lập để cộng mẫu; báo dải, khoảng tin cậy từng seed và dùng seed xấu nhất theo cổng của bài.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Theo `evidence/agents.json`, chọn `T=1,4 s`, W1 khoảng cách thời gian bằng 0,351877 so với 0,860401 của xe không phản ứng; trung vị người lái là 1,849474 s.
Xe sau dùng khoảng cách ban đầu 16 m và trễ phản ứng 0,6 s; bảng dưới lấy độ trễ xe thử 0,6 s, tỷ lệ là phần trăm số ô có va chạm trước hoặc sau.

| Bộ điều khiển | Xe sau phát lại: bất kỳ (%) | Xe sau phản ứng: bất kỳ (%) | Va chạm sau: phát lại / phản ứng (%) |
|---|---|---|---|
| constant-velocity | 73,370 | 73,370 | 0 / 0 |
| IDM | 70,074 | 5,148 | 64,926 / 0 |
| EgoMLP | 69,481 | 69,481 | 0 / 0 |
| PercMLP | 75,222 | 16,593 | 59,519 / 0 |

Xe sau phát lại giữ tốc độ cũ dù xe thử đã phanh, nên có thể đâm đuôi và tạo lỗi do thế giới không phản ứng; IDM bị ghi 64,926% va chạm sau trong thế giới này.
Khi xe sau phản ứng, va chạm sau IDM bằng 0; phần còn lại 5,148% trùng vùng bất khả thi, thay vì kết luận mọi cú đâm đều do IDM.
IDM tốt hơn constant-velocity ở cả hai thế giới trong bộ số này, nhưng chênh lệch tăng từ 3,296 lên 68,222 điểm phần trăm; PercMLP đổi từ tệ hơn constant-velocity sang tốt hơn.
W1 nhỏ hơn chỉ xác nhận phân phối khoảng cách thời gian gần log hơn trong phép hiệu chỉnh này, chưa chứng minh tài xế mô phỏng giống người thật trong mọi tình huống.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

`evidence/gate.json` chặn PercMLP ở R2–R6: cận trên Wilson toàn log 2,0425% vượt IDM 0,0347% + 1 điểm; cận trên lát phanh 8,8091% vượt ngưỡng R3 là 5%.
R5 trượt vì dải va chạm 0,3469–1,5262% rộng 1,1793 điểm; đây cùng R2/R3 là biểu hiện sai số tích lũy và bất ổn khi mô hình tự đưa mình vào trạng thái khác log huấn luyện.
R4 có 11,4444% lưới đâm dù tránh được, vượt mức 2%; R6 có 16,5926% va chạm trong thế giới phản ứng so với IDM 5,1481% + 2 điểm, gắn với các tình huống cắt vào ít có trong log.
Theo `evidence/fix.json`, giữ đúng 20.000 kịch bản cắt vào và 2 vòng DAgger: dữ liệu cắt vào bổ sung miền thiếu, còn DAgger gán nhãn dự báo IDM tại chính trạng thái mô hình tự đi vào để học phục hồi.
Sau sửa, R2–R6 đều chuyển sang đạt: cận trên toàn log 0,4533%, cận trên lát phanh 2,1601%, vùng đâm tránh được 0,1852%, dải va chạm 0,1734–0,2081% và va chạm với xe sau phản ứng 5,3333%; đủ 7/7 quy tắc nên cổng trả RELEASE.
L2 trung bình lại tăng từ 0,619818 lên 0,831669 m, cho thấy điểm bắt chước vòng hở và an toàn vòng kín không đồng nghĩa; PercMLP+ học nhãn IDM nên chưa có cơ sở kết luận vượt IDM.
Trên log, PercMLP+ vẫn đâm 5–6/2.883 ca so với IDM 1/2.883; ở cruise giữ lại 20 m/s, vùng đâm tránh được là 0,0741–1,1111% so với IDM 0%, nên chỉ kết luận đạt tiêu chí không kém hơn trong cổng đã chọn.
Chấp nhận RELEASE trong phạm vi kiểm thử của bài, chưa thay IDM trên xe thật; cần xác minh các ca còn đâm, dữ liệu độc lập, tốc độ ngoài miền huấn luyện và thử nghiệm có giám sát trước quyết định triển khai.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Thứ nhất, mô phỏng chỉ xét chuyển động dọc: lưới cruise 10 m/s với 2.700 ô không kiểm tra lái ngang, giao lộ hoặc người đi bộ; cần bổ sung tương tác nhiều chiều trước khi suy rộng sang giao thông thực.
Thứ hai, mô hình nhận trạng thái gần như trực tiếp; phép nhiễu khoảng cách 1 m và tốc độ xe dẫn 0,5 m/s trong `closedloop.json[noise]` chưa bao phủ bỏ sót vật thể, che khuất và trễ cảm biến có tương quan; cần thử cả chuỗi cảm nhận đến điều khiển.
Thứ ba, xe dẫn trong log phát lại không phản ứng, còn xe sau chỉ là IDM được hiệu chỉnh `T=1,4 s`; việc va chạm sau IDM đổi từ 64,926% xuống 0 cho thấy kết luận phụ thuộc mô hình tác nhân, cần kiểm tra nhiều kiểu phản ứng và đối chiếu log tương tác thực.
Thứ tư, 2.883 quỹ đạo giữ lại, 174 ca có xe dẫn phanh và 3 seed còn hạn chế: CI95 và số ô phủ không đo được các sự kiện hiếm ngoài mẫu; cần đánh giá thêm dữ liệu độc lập, miền vận hành và các điều kiện khó.
Cuối cùng, cơ cấu chấp hành bị giản lược bằng giới hạn 9 m/s² và bước thời gian cố định; cần kiểm chứng trên mô phỏng có ma sát, trễ phanh và phần cứng thực tế rồi thử trong khu vực kiểm soát.
Do đó RELEASE của cổng chỉ là đạt tiêu chí của bài lab trên tập thử này; chưa đủ bằng chứng cho quyết định triển khai xe thật.
