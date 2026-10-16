# REPORT — Lab Day 12

> Bằng chứng được sinh trực tiếp từ pipeline với MSSV `2A202602415`, đủ 5 seed và không dùng chế độ quick. Các tỷ lệ phần trăm dưới đây được quy đổi từ JSON; chênh lệch tỷ lệ ghi bằng điểm phần trăm.

| | |
|---|---|
| Họ và tên | `Nguyễn Hồng Cường` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602415` |
| GitHub | `hongcuong26-debug` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `8656 / 3697 (các seed 3697–3701)` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5 (không dùng --quick)` |

## Kết quả cuối

Kết quả từ `evidence/openloop.json`, `closedloop.json` và `gate.json`; dải là min–max qua 5 seed. Pipeline gốc chỉ lưu L2 trung bình/độ lệch chuẩn của PercMLP+, nên `evidence/openloop_fixed_per_seed.json` được sinh tự động bằng cách chấm lại đúng 5 checkpoint sau sửa trên cùng tập test, không huấn luyện lại hoặc sửa JSON gốc.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.275526 | 20.083247 | BLOCK |
| IDM | 1.526611 | 0.138744 | Baseline (không xét ứng viên) |
| EgoMLP | 0.764198–0.776077 | 15.053763–31.772459 | BLOCK |
| PercMLP | 0.618473–0.637129 | 0.554977–5.584461 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.793774–0.826394 | 0.173430–0.242803 | RELEASE (7/7) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Theo `evidence/data_audit.json`, dữ liệu có 992679 dòng, 9993 quỹ đạo thô và còn 9610 quỹ đạo sau khi loại 383 bản trùng.
Tương quan của `Acc_LV` với đạo hàm vận tốc chỉ 0.045354, trong khi cột của xe theo sau đạt 0.929087; vì vậy không thể tin gia tốc xe dẫn ghi sẵn.
Tôi dùng `a_lead`, được suy ra từ `Speed_LV` sau khi làm mượt, để khai thác phanh; các quỹ đạo dài 70–109 bước, lấy mẫu mỗi 0.1 s.
Khoảng cách khớp hiệu vị trí trừ chiều dài xe 4.5 m, với sai lệch cực đại chỉ khoảng 3.64e-11 m; đây là kiểm tra tính nhất quán hình học.
Cần loại trùng trước khi chia theo quỹ đạo, vì ID khác vẫn có thể là cùng một chuỗi lái; nếu chia theo khung, cửa sổ gần nhau còn dễ rò rỉ sang test.
Chia nguyên quỹ đạo giúp tập test kiểm tra chuỗi lái chưa thấy, thay vì chỉ kiểm tra các khung lân cận đã học.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Trong `evidence/replay.json`, chạy lại cùng mã cho 0/600 đoạn khác; refactor float32 không giống hệt ở 600/600 đoạn nhưng có 0 đoạn vượt dung sai.
Sai khác tuyệt đối lớn nhất của refactor là 9.733409e-05 m/s²; dung sai dùng `atol=0.001`, `rtol=0.001`, theo trị tuyệt đối của lệnh chuẩn.
Thêm AEB gây 8/600 đoạn vượt dung sai (9 đoạn không giống hệt), vì AEB chỉ đổi lệnh khi tốc độ tiến gần dương, TTC dưới 1.5 s và lệnh cũ chưa phanh đến -6 m/s².
Tôi mở đoạn 73 trong `work/replay_log.jsonl`: khung 73 có gap 3.7574 m, closing 2.7173 m/s, TTC 1.38277 s; lệnh đổi từ -5.58041 sang -6.0 m/s².
Phát nhanh nhất ghi 0 khung trễ và 0 khung rớt; phát theo thời gian thật có 582 khung trễ, 576 khung rớt trong 59745 khung, làm 372 đoạn vượt dung sai.
Chi phí xử lý ở đây được mô phỏng: chế độ thời gian thật phơi lộ vi phạm hạn và việc giữ lệnh cũ, còn phát nhanh nhất không áp đặt hạn nên che lỗi đó.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Theo `evidence/scenarios.json`, tốc độ cruise là 10 m/s, gap từ 2.760051 đến 59.312326 m; lưới gồm 60 × 45 = 2700 ô, closing từ 0 đến 10 m/s.
Biên bất khả thi là `gap < dv*t_react + dv²/(2*b_max)`, với b_max = 9 m/s²; phía trong biên thì ngay cả phanh tối đa cũng không tránh được.
Khi độ trễ tăng từ 0 lên 0.6 s, vùng bất khả thi tăng từ 0.962963% lên 5.148148%; vùng va chạm IDM cũng tăng đúng các mức này, tức 4.185185 điểm phần trăm.
IDM có diện tích va chạm quy trách nhiệm bằng 0 ở cả hai độ trễ; kiểm chứng FullBrake cho 0 ô lệch xa biên, nên không quy lỗi bộ điều khiển cho các ô vật lý không cứu được.
Log cho 614 sự kiện phanh, chiếm 6.389178% quỹ đạo; giới hạn phanh người lái tại phân vị 99.9% là 3.624837 m/s², thấp hơn giới hạn cơ cấu chấp hành.
Có 5.222222% lưới đòi phanh mạnh hơn mức đó; các ca phanh tự nhiên không phủ đủ vùng tiến gần nhanh, gap hẹp, nên điểm tốt trên log chưa chứng minh xử lý tốt cắt đầu.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng dưới dùng `openloop.json.summary` và `closedloop.json.ranking`: tỷ lệ vòng kín để xếp hạng là trung bình qua seed, khác dải và seed xấu nhất dùng ở cổng.

| Bộ điều khiển | L2 trung bình (m) | Hạng vòng hở | Va chạm trung bình (%) | Hạng vòng kín |
|---|---:|---:|---:|---:|
| constant-velocity | 1.275526 | 3 | 20.083247 | 3 |
| IDM | 1.526611 | 4 | 0.138744 | 1 |
| EgoMLP | 0.768314 | 2 | 25.140479 | 4 |
| PercMLP | 0.623973 | 1 | 1.977107 | 2 |

Bốn cặp đảo thứ tự là constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP.
IDM không tối ưu bắt chước vị trí người lái; dự báo vòng hở còn giả định xe dẫn giữ tốc độ, nên L2 cao trong khi luật khoảng cách và phanh giúp nó phản ứng tốt khi tự lái.
EgoMLP giảm L2 nhờ học mẫu tốc độ, nhưng thiếu gap và tốc độ xe dẫn nên không phân biệt cùng lịch sử ego mà nguy cơ phía trước khác nhau.
Khi tự lái, sai số đầu ra làm trạng thái sau rời phân phối log và tiếp tục tích lũy; L2 thấp trên cửa sổ thật không đảm bảo hành động an toàn trên trạng thái do mô hình tạo ra.
Lát cắt phanh chỉ chiếm 4.632568% cửa sổ test; L2 PercMLP tăng từ 0.623973 m toàn bộ lên 0.906132 m ở lát cắt, cho thấy trung bình vẫn che các trường hợp khó.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Theo `evidence/closedloop.json`, EgoMLP va chạm từ 15.053763% (seed 3701) đến 31.772459% (seed 3700), chênh 16.718696 điểm phần trăm dù cùng tập 2883 quỹ đạo test.
Seed tốt nhất có 434 ca va chạm và khoảng Wilson 95% [13.794953%, 16.405581%]; seed xấu nhất có 916 ca và khoảng [30.098103%, 33.495327%].
Hai khoảng không chồng nhau; biến động do huấn luyện lớn hơn nhiều độ bất định tỷ lệ trong từng lần đánh giá, nên không được lấy một seed thuận lợi đại diện cho mô hình.
Nếu chỉ báo seed 3701, EgoMLP có thể trông tốt hơn constant-velocity (20.083247%); chọn seed 3700 thì kết luận ngược lại, trong khi trung bình năm seed là 25.140479%.
Wilson mô tả độ bất định của tỷ lệ theo mẫu quỹ đạo dưới giả định thống kê, không phải khoảng biến động qua seed và cũng không bảo đảm an toàn ngoài phân phối.
Tôi báo cả dải 5 seed, khoảng tin cậy và dùng seed xấu nhất tại cổng; còn cần thêm dữ liệu độc lập vì nhiều quỹ đạo giao thông có thể tương quan.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng lấy từ `evidence/agents.json.rear_comparison_t_react_0.6`, đơn vị là phần trăm ô lưới có ít nhất một va chạm; mô hình học dùng seed xấu nhất theo từng chế độ.

| Bộ điều khiển | Xe sau phát lại: any (%) | Xe sau phản ứng: any (%) | Va chạm sau: phát lại → phản ứng (%) |
|---|---:|---:|---:|
| constant-velocity | 73.370370 | 73.370370 | 0 → 0 |
| IDM | 70.074074 | 5.148148 | 64.925926 → 0 |
| EgoMLP | 75.111111 | 75.111111 | 0 → 0 |
| PercMLP | 72.740741 | 31.037037 | 44.407407 → 0 |

Hiệu chỉnh chọn T = 1.4 s, cho khoảng cách Wasserstein-1 giữa phân phối headway là 0.341670, tốt hơn tác nhân không phản ứng 0.824526; headway trung vị người lái là 1.853340 s.
T cho W1 nhỏ nhất trong tập thử giúp xe sau giống phân phối quan sát hơn, nhưng không chứng minh phản ứng của từng tài xế hoặc mọi tình huống đều chân thực.
Khi IDM phanh tránh xe cắt đầu, xe sau phát lại vẫn giữ chuyển động định trước nên đâm đuôi ở 64.925926% ô; khi biết phản ứng, phần này về 0 và any chỉ còn 5.148148%.
Ở độ trễ 0 s, IDM còn đổi từ 69.555556% any xuống 0.962963%; đó là bằng chứng loại tác nhân có thể làm thay đổi mạnh đánh giá dù chính bộ điều khiển ego không đổi.
Trong bộ số này IDM vẫn tốt hơn constant-velocity ở cả hai thế giới (70.074074% so với 73.370370% khi phát lại), nên không có đảo hạng giữa hai bộ này; thế giới phản ứng làm rõ ưu thế lớn vốn bị va chạm giả che khuất.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Theo `evidence/gate.json`, PercMLP trượt R2, R3, R4, R5 và R6: cận trên va chạm log 6.483% vượt ngưỡng 1.139%; cận trên lát cắt phanh 28.5% vượt ngưỡng 7.1%; vùng cắt đầu quy trách nhiệm 25.889% vượt 2%.
R5 phản ánh dải va chạm 0.554977–5.584461% (chênh 5.029484 điểm phần trăm), còn R6 có 31.037037% ô va chạm trong thế giới phản ứng so với IDM 5.148148%; đây là bằng chứng cho sai số tích lũy và thiếu phủ kịch bản, chưa phải phép chứng minh nhân quả riêng lẻ.
Tôi bổ sung 20000 kịch bản cắt đầu (197100 cửa sổ) để bù vùng log không chứa; 2 vòng DAgger/seed lấy chính trạng thái mô hình tự đi vào rồi gắn nhãn dự báo 3 s của IDM, giúp học cách phục hồi từ sai lệch.
Sau sửa, cả R1–R7 đều qua: va chạm log còn 0.173430–0.242803%, cận trên seed xấu nhất 0.500369%, cận trên lát cắt phanh 5.287250%, vùng cắt đầu quy trách nhiệm 0.148148% và va chạm trong thế giới phản ứng 5.296296%.
L2 trung bình tăng từ 0.623973 lên 0.813768 m dù an toàn vòng kín tốt hơn; đó là lý do không chọn mô hình chỉ bằng mức bắt chước log. PercMLP+ vẫn có va chạm log cao hơn IDM 0.138744%, nên qua cổng chỉ chứng tỏ đạt tiêu chí không-kém-hơn với biên đã chọn.
Tại cruise giữ lại 20 m/s (huấn luyện cắt đầu chỉ tới 16 m/s), `evidence/fix.json` cho PercMLP+ còn 0.296296–0.888889% ô tránh được nhưng vẫn đâm, trong khi IDM là 0%; chưa có bằng chứng mô hình học vượt chuyên gia IDM.
Tôi chấp nhận RELEASE trong phạm vi cổng bài lab, chưa đề nghị thay IDM trên xe thật; cần đánh giá độc lập ngoài miền, thêm nhiều kiểu tác nhân, lỗi cảm biến/chấp hành và đường thử có giám sát trước quyết định triển khai.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Thứ nhất, mô phỏng chỉ mô tả chuyển động dọc: lưới 2700 ô ở cruise 10 m/s không bao quát đánh lái, hình học làn, đường cong hay tương tác tại giao lộ, nên qua cổng chưa đủ để quyết định phát hành xe thật.
Thứ hai, quan sát và cơ cấu chấp hành còn lý tưởng hóa; dù có phép thử nhiễu gap 1 m, tốc độ xe dẫn 0.5 m/s và độ trễ 0.6 s, chúng chưa mô tả mất phát hiện, sai liên kết đối tượng, ma sát thấp hoặc lỗi phanh.
Thứ ba, xe dẫn trong log không phản ứng với ego; xe sau phản ứng cũng chỉ là IDM hiệu chỉnh T = 1.4 s với W1 = 0.341670, nên lựa chọn mô hình thế giới có thể làm lệch kết luận an toàn.
Thứ tư, tập dữ liệu chỉ còn 9610 quỹ đạo ngắn 70–109 bước, còn 614 sự kiện phanh; các ca hiếm, khác miền và tương quan giữa quỹ đạo khiến Wilson 95% không phải bảo đảm rủi ro ngoài thực tế.
Thứ năm, PercMLP+ dùng nhãn chuyên gia IDM và cổng chủ yếu đo không-kém-hơn IDM trong các phép thử hữu hạn; điều đó chưa đo đầy đủ độ êm, tiến độ hành trình, tính công bằng hay khả năng vượt baseline.
Trước phát hành cần dữ liệu và kịch bản độc lập ngoài miền, mô phỏng đa tác nhân nhiều chiều, cảm biến và chấp hành có lỗi được hiệu chuẩn, rồi kiểm thử phần cứng và đường thử có giám sát; chỉ nâng mức triển khai khi có bằng chứng mới.
