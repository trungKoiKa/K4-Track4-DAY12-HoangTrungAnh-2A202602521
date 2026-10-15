# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Hoàng Trung Anh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602521 |
| GitHub | trungKoiKa |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 8606 / 4150 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (mặc định) |

## Kết quả cuối

Số lấy từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (mô hình học: dải qua 5 seed; cổng dùng seed xấu nhất).

| Bộ điều khiển | L2 vòng hở (m), toàn bộ / lát phanh | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,261 / 1,931 | 19,9 | BLOCK (R2, R3, R4, R6) |
| IDM | 1,557 / 1,712 | 0,1 | (baseline) |
| EgoMLP | 0,764 / 1,056 | 9,4 .. 28,5 (trung bình 18,2) | BLOCK (R2–R6) |
| PercMLP | 0,626 / 0,901 | 0,8 .. 4,6 (trung bình 2,1) | BLOCK (R2–R6) |
| PercMLP+ (sau khi sửa) | 0,830 / 1,051 | 0,3 .. 0,3 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 cho thấy ba vấn đề. (1) Cột `Acc_LV` vô dụng: tương quan với đạo hàm `Speed_LV` chỉ 0,045, trong khi cột của xe theo sau là 0,929. Mình thay bằng gia tốc xe dẫn tính từ tốc độ đã làm mượt (`a_lead`) và dùng nó để khai thác ca phanh. (2) Có trùng lặp: 383 quỹ đạo giống hệt nhau nằm ở các id cách xa nhau, nên từ 9.993 còn 9.610 quỹ đạo sạch. (3) Cột khoảng cách lệch một hằng số: `gap = pos_lead - pos_ego - 4,5 m` (độ dài xe), sai lệch tối đa chỉ 3,6e-11 m. Phải chia theo quỹ đạo vì các cửa sổ trong cùng một quỹ đạo gần như giống nhau; chia theo khung hình sẽ để train và test chứa cùng một chuyến đi, các bản trùng còn làm rò rỉ thêm, nên điểm test bị thổi phồng.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 không giống từng bit ở 600/600 đoạn (lệch tối đa 1,2e-4) nhưng 0 đoạn vượt dung sai (atol = rtol = 1e-3). So bằng `==` là sai vì số thực dấu phẩy động không có tính kết hợp: (0,1+0,2)+0,3 = 0,6000000000000001 còn 0,1+(0,2+0,3) = 0,6; chỉ dung sai mới phân biệt được "khác bit" với "khác hành vi". Thay đổi AEB làm lệch 8/600 đoạn. Mở đoạn 182: chỉ 2 khung khác (khung 87, 88), TTC 1,49 và 1,46 s; đoạn 250 có 7 khung, TTC 1,36–1,46 s. Cả hai đều dưới ngưỡng AEB 1,5 s và lệnh đổi từ khoảng −4,2 .. −5,9 sang −6,0 m/s². Ở các khung còn lại TTC lớn hơn 1,5 nên AEB không kích hoạt và lệnh không đổi. Phát nhanh nhất có 0 khung trễ, 0 khung rớt vì không có hạn chót; phát theo thời gian thật có 582 khung trễ và 574 khung rớt trên 59.724 khung, làm 377/600 đoạn thay đổi (lệch tối đa 50,3). Phát nhanh nhất che đi lỗi trễ hạn chót này.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới gồm 60 khoảng cách × 45 tốc độ tiến gần = 2.700 ô (tốc độ cruise 12 m/s). Với độ trễ 0 s, 2,2 % ô bất khả thi (không ai tránh được); với độ trễ 0,6 s là 7,4 %. Vùng đâm của IDM tăng từ 2,2 % lên 7,4 % (gấp khoảng 3,4 lần) và khớp đúng biên vật lý, vì kiểm chứng cho 0 ô lệch ở cả hai độ trễ. Nghĩa là IDM chỉ đâm ở nơi không ai tránh được (phần "tránh được mà vẫn đâm" của IDM là 0,0 %), còn constant-velocity đâm 70,2 % và EgoMLP 72,8 % số ô lẽ ra tránh được (độ trễ 0,6 s, seed xấu nhất). Log không phủ vùng khó: người lái trong log phanh mạnh nhất chỉ khoảng 3,6 m/s² (phân vị 99,9) trong khi cơ cấu phanh làm được 9 m/s², và 9 % lưới đòi phanh mạnh hơn mức đó. Trong 9.610 quỹ đạo chỉ có 614 ca xe dẫn phanh (6,4 %).

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?


| Bộ điều khiển | L2 (m) | Hạng L2 | Va chạm (%) | Hạng va chạm |
|---|---|---|---|---|
| const-vel | 1,261 | 3 | 19,9 | 4 |
| IDM | 1,557 | 4 | 0,1 | 1 |
| EgoMLP | 0,764 | 2 | 18,2 | 3 |
| PercMLP | 0,626 | 1 | 2,1 | 2 |

Các cặp đảo thứ hạng: (const-vel, IDM), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất vì dự báo 3 giây của nó giả định xe dẫn giữ nguyên tốc độ, trong khi người lái thật phản ứng khác; nhưng nó là bộ điều khiển phản hồi: mỗi 0,1 s nhìn lại khoảng cách và tốc độ thật rồi sửa lệnh, nên sai số dự báo không tích luỹ. Mô hình học được chấm trên trạng thái của người lái, nhưng khi tự lái nó đi vào trạng thái người chưa từng gặp và sai số cộng dồn. EgoMLP chỉ thấy tốc độ của chính nó, không thấy khoảng cách và xe dẫn, nên chỉ học được mức giảm tốc trung bình; nó không biết xe dẫn đang phanh và đâm 61,7 % số ca xe dẫn phanh (seed xấu nhất). L2 chỉ đo sai số trung bình trên log; lát phanh chỉ chiếm 4,7 % cửa sổ nên che khuất đúng nơi gây va chạm.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Va chạm của EgoMLP dao động từ 9,4 % đến 28,5 % qua 5 seed (15,3; 15,4; 28,5; 22,4; 9,4). Khoảng Wilson 95 % của seed xấu nhất là [26,9; 30,2] %, còn seed tốt nhất 9,4 % nằm cách xa nên hai khoảng không chồng nhau: chênh lệch đến từ seed huấn luyện chứ không phải nhiễu lấy mẫu. Nếu chỉ chạy một seed, mình có thể báo EgoMLP đâm 9,4 % (nghe như tạm ổn) hoặc 28,5 % (thảm hoạ) tuỳ may rủi, và không có cách nào biết mình đang ở đâu. PercMLP cũng dao động 0,8 .. 4,6 %. Đó là lý do cổng có R5 (ổn định qua seed) và dùng seed xấu nhất; EgoMLP rớt R5 vì độ rộng khoảng 19 điểm so với giới hạn 1 điểm.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng với độ trễ 0,6 s (tỷ lệ ô có va chạm bất kỳ, trước hoặc sau; cột cuối: riêng va chạm phía sau):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 77,6 % | 77,6 % | 0,0 / 0,0 |
| IDM | 77,4 % | 7,4 % | 70,0 / 0,0 |
| EgoMLP | 80,3 % | 80,3 % | 0,0 / 0,0 |
| PercMLP | 77,8 % | 31,9 % | 47,4 / 0,0 |

Mình hiệu chỉnh `T = 1,2 s` (khoảng cách phân bố W1 = 0,34); xe sau không bao giờ phản ứng có W1 = 0,87, nên xe sau IDM này chân thực hơn rõ rệt (log có thời khoảng cách trung vị 1,84 s). Khi xe sau chỉ phát lại, nó cứ giữ tốc độ log dù xe thử vừa phanh gấp, nên IDM bị tính 70,0 % ô "va chạm sau" (PercMLP 47,4 %). Đó là va chạm giả: xe sau thật sẽ phản ứng, và khi cho nó phản ứng thì va chạm sau về 0,0 % và IDM chỉ còn 7,4 %, đúng bằng phần bất khả thi vật lý. Constant-velocity và EgoMLP không phanh nên không bị đâm từ phía sau, và trông "ngang" IDM (77–80 % so với 77,4 %) trong thế giới phát lại; chỉ khi loại tác nhân phát lại mới thấy IDM tốt hơn hẳn (7,4 % so với 77,6 %). Kết luận về bộ điều khiển đổi theo giả định về thế giới.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi R2 (4,6 %, cận trên 5,4 % so với IDM 0,1 % + 1 điểm), R3 (lát phanh 15,3 % so với 1,0 % + 5 điểm), R4 (24,5 % lưới so với 0,0 % + 2 điểm), R5 (0,8 .. 4,6 % qua 5 seed) và R6 (31,9 % so với 7,4 %). Có hai nguyên nhân gốc. (a) Kịch bản log không chứa: log không có ca cắt vào và phanh mạnh, nên lưới cắt vào (R4, R6) rớt 24,5 % và 31,9 %; thử thêm tốc độ cruise 20 m/s (train chỉ tới 16) PercMLP đâm 25,7 .. 31,9 % lưới. (b) Sai số tích luỹ khi mô hình tự lái: ngay trên log, R2/R3/R5 rớt vì va chạm 0,8 .. 4,6 % dù L2 tốt nhất. Mình sửa bằng hai cách. Một là thêm 197.210 cửa sổ từ 20.000 cắt vào mô phỏng với nhãn IDM, vá nguyên nhân (a) vì đưa đúng vùng còn thiếu vào huấn luyện. Hai là DAgger 2 vòng, huấn luyện lại ở các trạng thái mô hình tự đi vào với nhãn IDM, vá nguyên nhân (b). Kết quả PercMLP+ qua cả 7 quy tắc: va chạm 0,3 % (cận trên 0,6 %), lát phanh 2,0 %, lưới 0,1 %, tốc độ 20 m/s còn 0,0 .. 0,6 %, vùng phản ứng 7,6 % so với IDM 7,4 %. Nó chỉ **không kém hơn** IDM (trong biên 1 và 2 điểm), **không tốt hơn**: 0,3 % so với 0,1 % và 7,6 % so với 7,4 %; nó học nhãn từ IDM nên khó vượt thầy. L2 vòng hở của PercMLP+ (0,830 m) lại tệ hơn PercMLP (0,626 m) dù lái an toàn hơn nhiều, tức điểm vòng hở không phản ánh chất lượng lái. Mình không đề xuất thay IDM ngay vì chưa có lợi ích rõ so với IDM: trước hết cần thêm kịch bản (tốc độ, độ trễ, nhiễu cảm biến khác), chạy bóng ma (shadow mode: chạy song song nhưng không điều khiển) trên xe thật, và chỉ thay nếu có lợi ích mà IDM không có.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) Một chiều dọc, không có cảm biến thật: không có làn, xe cắt vào bên cạnh, đường cong, nhiễu và độ trễ thật; PercMLP+ qua cổng không nói gì về các tình huống đó. (2) Xe dẫn trong log không phản ứng với xe thử, còn xe sau chỉ là IDM mô phỏng với `T = 1,2 s` (W1 = 0,34, chưa khớp hẳn); toàn bộ kết luận về va chạm "giả" hay thật phụ thuộc giả định đó. (3) Thế giới mô phỏng, nhãn và ngưỡng cổng do chính chúng ta chọn (lưới cắt vào, biên 1/5/2 điểm, IDM làm thầy); PercMLP+ qua cổng một phần vì được huấn luyện trên chính họ kịch bản mà cổng kiểm tra, nên điểm cao có thiên lệch, và tốc độ 20 m/s mới là phép thử ngoài phân phối. (4) Dữ liệu chỉ từ một bộ (Argoverse 2, người lái ở đô thị), 614 ca phanh, log không phủ vùng phanh trên 3,6 m/s². Để thu hẹp khoảng cách: kiểm thử với xe khác phản ứng thật, thêm kịch bản ngoài phân phối, nhiều seed hơn, và chạy bóng ma trên đường thật trước khi quyết định phát hành.
