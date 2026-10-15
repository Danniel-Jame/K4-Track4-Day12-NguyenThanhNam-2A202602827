# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Nguyễn Thành Nam` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602827` |
| GitHub | `https://github.com/Danniel-Jame/K4-Track4-Day12-NguyenThanhNam-2A202602827` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `2901/8205` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.28 | 20.57% | Bị chặn (R2, R3, R4, R6) |
| IDM | 1.54 | 0.00% | Đạt (Baseline) |
| EgoMLP | 0.77 (dải 0.767 .. 0.770) | 26.07% (dải 22.44% .. 30.70%) | Bị chặn (R2, R3, R4, R5, R6) |
| PercMLP | 0.63 (dải 0.628 .. 0.636) | 0.54% (dải 0.35% .. 0.87%) | Bị chặn (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.65 (dải 0.645 .. 0.652) | 0.10% (dải 0.00% .. 0.17%) | Đạt (Đã vượt cổng) |

## Câu 1 · Dữ liệu

Trong Bước 0, dữ liệu gốc có 9,993 quỹ đạo với 992,679 dòng. Em phát hiện cột `Acc_LV` (gia tốc xe dẫn) có hệ số tương quan với đạo hàm tốc độ gần như bằng 0 ($0.0454$), tức là số liệu cột này bị sai lệch nặng. Trong khi đó, gia tốc xe sau có tương quan rất cao ($0.9291$), nên em thay gia tốc xe dẫn bằng đạo hàm đã làm mượt của `Speed_LV`.

Tập dữ liệu có 383 quỹ đạo bị trùng lặp hoàn toàn và đã bị loại bỏ, còn lại 9,610 quỹ đạo sạch. Việc chia train/test bắt buộc phải chia theo từng quỹ đạo thay vì chia theo khung hình (frame), vì các khung hình kề nhau trong cùng một chuyến đi có tính tương quan rất cao; nếu chia ngẫu nhiên theo khung hình sẽ gây rò rỉ dữ liệu (data leakage) giữa tập train và test.

## Câu 2 · Phát lại

Bản refactor float32 cho kết quả khác biệt từng bit ở toàn bộ 600 đoạn do sai số làm tròn số thực, nhưng không có đoạn nào vượt dung sai $0.001\text{ m/s}^2$ (`differ_beyond_tol` = 0, sai lệch tối đa $9.73 \times 10^{-5}\text{ m/s}^2$). Khi bật AEB với $TTC = 1.5\text{ s}$, có đúng 8 đoạn bị lệch vượt dung sai (đoạn 62, 128, 162, 182, 188, 305, 523, 547).

Mở đoạn 128, lệnh bị lệch ở đúng các khung hình xe sau tiến quá gần xe dẫn làm chỉ số TTC giảm xuống dưới $1.5\text{ s}$, kích hoạt AEB phanh khẩn cấp để đè lên lệnh phanh thường. Cuối cùng, khi phát theo thời gian thật (`realtime`), hệ thống bị trễ 580 khung và rớt 574 khung, khiến 374 đoạn bị lệch. Chế độ phát nhanh nhất (`fastest`) hoàn toàn che giấu lỗi trễ khung và rớt khung này vì nó không tính thời gian thực thi.

## Câu 3 · Kịch bản

Lưới kịch bản cắt vào được tạo với tốc độ cruise $v_{ego} = 12.0\text{ m/s}$, khoảng cách $gap$ từ $2.76\text{ m}$ đến $59.31\text{ m}$, và tốc độ tiến gần $dv$ từ $0.0$ đến $12.0\text{ m/s}$ (gồm 2,700 ô). Khi độ trễ phản ứng $t_{react} = 0.0\text{ s}$, có $2.22\%$ số ô trên lưới là va chạm bất khả thi (không xe nào tránh được). Khi tăng độ trễ phản ứng $t_{react}$ từ $0.0\text{ s}$ lên $0.6\text{ s}$, vùng đâm bất khả thi của IDM tăng thêm $5.22$ điểm %, đạt $7.44\%$ lưới.

Log thực tế không phủ được các vùng tình huống khó. Mức phanh mạnh nhất của người lái trong log ($99.9\text{th}$ percentile) chỉ đạt $3.62\text{ m/s}^2$, và có $8.56\%$ diện tích lưới đòi hỏi lực phanh mạnh hơn mức này. Ngoài ra, 614 sự kiện phanh khai thác từ log chỉ có mức phanh trung vị là $2.53\text{ m/s}^2$, chứng tỏ người thật trong log hiếm khi phanh sát biên nguy hiểm.

## Câu 4 · Vòng hở và vòng kín

* **Bảng L2 vòng hở (m):** PercMLP ($0.63\text{ m}$, Hạng 1) < EgoMLP ($0.77\text{ m}$, Hạng 2) < constant-velocity ($1.28\text{ m}$, Hạng 3) < IDM ($1.54\text{ m}$, Hạng 4).
* **Bảng Tỷ lệ va chạm vòng kín (%):** IDM ($0.00\%$, Hạng 1) < PercMLP ($0.54\%$, Hạng 2) < constant-velocity ($20.57\%$, Hạng 3) < EgoMLP ($26.07\%$, Hạng 4).
* **Các cặp đổi thứ tự xếp hạng:**
  1. `[const-vel, IDM]`: Vòng hở const-vel tốt hơn IDM ($1.28\text{ m}$ vs $1.54\text{ m}$), nhưng vòng kín IDM lái an toàn tuyệt đối ($0.00\%$) còn const-vel va chạm tới $20.57\%$.
  2. `[const-vel, EgoMLP]`: Vòng hở EgoMLP điểm tốt hơn const-vel ($0.77\text{ m}$ vs $1.28\text{ m}$), nhưng vòng kín EgoMLP đâm nhiều hơn ($26.07\%$ vs $20.57\%$).
  3. `[IDM, EgoMLP]` & `[IDM, PercMLP]`: IDM đứng chót ở điểm vòng hở nhưng đứng đầu ở vòng kín.

Luật IDM có L2 vòng hở kém vì nó không bắt chước thói quen lái riêng của từng người trong log, nhưng khi vào vòng kín, nó dùng phản hồi cự ly $gap$ để điều chỉnh phanh liên tục nên an toàn $100\%$. Trái lại, EgoMLP có L2 vòng hở rất đẹp nhờ nhớ quy luật tốc độ bản thân, nhưng vì hoàn toàn không nhìn thấy khoảng cách xe trước, khi gặp lát cắt xe trước phanh gấp (L2 lát cắt phanh lên tới $1.07\text{ m}$), nó bị "mù" và tích lũy sai số gây ra tỷ lệ va chạm rất cao ($26.07\%$).

## Câu 5 · Một lần chạy chưa đủ

Mô hình EgoMLP dao động rất mạnh qua 3 seed huấn luyện: Seed 8205 đạt $22.44\%$ va chạm (KTC 95%: $[20.96\%, 24.00\%]$), Seed 8207 đạt $25.08\%$ ($[23.53\%, 26.69\%]$), và Seed 8206 vọt lên $30.70\%$ ($[29.04\%, 32.41\%]$). Mức chênh lệch giữa seed tốt nhất và xấu nhất lên tới $8.26$ điểm %.

Khoảng tin cậy 95% Wilson của seed 8205 ($[20.96\%, 24.00\%]$) và seed 8206 ($[29.04\%, 32.41\%]$) tách rời nhau hoàn toàn và không hề chồng lấp. Nếu chỉ chạy 1 seed duy nhất và gặp trúng seed may mắn (8205), ta sẽ báo cáo kết quả $22.44\%$ và lầm tưởng mô hình tốt. Điều này cực kỳ nguy hiểm vì nó che giấu sự mất ổn định do khởi tạo ngẫu nhiên, dẫn đến nguy cơ đưa một mô hình có thể đâm tới $30.70\%$ ra sử dụng.

## Câu 6 · Tác nhân phản ứng

Khi xe sau chỉ "phát lại" log cũ, với $t_{react} = 0.6\text{ s}$, IDM bị tính $75.74\%$ va chạm (trong đó $68.30\%$ là bị đâm từ phía sau) và PercMLP bị tính $83.15\%$ va chạm ($61.85\%$ va chạm phía sau). Khi chuyển sang "xe sau phản ứng" với $T = 1.4\text{ s}$, tỷ lệ va chạm phía sau của cả IDM lẫn PercMLP đều giảm về đúng $0.00\%$, làm tổng tỷ lệ va chạm giảm xuống còn $7.44\%$ đối với IDM và $21.93\%$ đối với PercMLP.

Khoảng thời gian cách $T = 1.4\text{ s}$ được hiệu chỉnh cho khoảng cách Wasserstein $W_1 = 0.3419$, chân thực hơn rất nhiều so với xe không phản ứng ($W_1 = 0.8546$). Lỗi "va chạm giả" xảy ra do xe sau phát lại không hề biết xe thử vừa phanh nên cứ lao thẳng tới đâm vào đuôi. Thay bằng tác nhân phản ứng giúp loại bỏ các tai nạn vô lý này và minh chứng rằng IDM thực sự lái rất an toàn.

## Câu 7 · Cổng phát hành và sửa

Mô hình PercMLP bị chặn bởi 4 quy tắc: R2 (cận trên va chạm vòng kín $1.3\% > 1.0\%$), R3 (cận trên lát cắt phanh $5.4\% > 5.0\%$), R4 (diện tích đâm trên lưới kịch bản $14.5\% > 2.0\%$), và R6 (diện tích va chạm trong thế giới phản ứng $21.9\% > 9.4\%$). Nguyên nhân gốc do dữ liệu log không chứa các pha tạt đầu ở cự ly gần làm mô hình bị ngợp (OOD), cộng với sai số dự báo tích lũy dần trong vòng kín.

Mô hình được sửa bằng cách thêm dữ liệu mô phỏng tạt đầu (`cutin_windows`) và áp dụng DAgger (`dagger_windows`) để thu thập các trạng thái lệch rồi lấy nhãn phanh từ IDM. Sau khi sửa, PercMLP+ giảm tỷ lệ đâm xuống còn $0.10\%$ và vượt qua toàn bộ các quy tắc cổng. Tuy nhiên, PercMLP+ không "tốt hơn" IDM vì nó chỉ học lại từ chính IDM. Ta không nên thay thế IDM ngay mà cần thử nghiệm thêm trong nhiều điều kiện phức tạp khác.

## Câu 8 · Bài lab này không chứng minh được gì

Ba giới hạn quan trọng nhất của bài lab gồm:
1. **Mô hình động học 1D quá đơn giản:** Chỉ xét chuyển động dọc trên một làn đường, bỏ qua hành vi đánh lái tránh vật cản, chuyển làn và góc trượt của lốp.
2. **Cảm biến hoàn hảo không trễ:** Giả định xe nhận được khoảng cách và tốc độ chính xác $100\%$ mà không có nhiễu cảm biến, thời tiết xấu hay trễ nhận dạng camera/LiDAR.
3. **Xe dẫn không phản ứng linh hoạt:** Xe phía trước di chuyển theo kịch bản cố định trong log chứ không tương tác qua lại theo hành vi của xe thử.

Để thu hẹp khoảng cách với thực tế, bước tiếp theo cần làm là đưa bộ điều khiển vào môi trường mô phỏng 3D vật lý đầy đủ (như CARLA), thêm nhiễu cảm biến thực tế, thử nghiệm tương tác đa tác nhân hai chiều và tiến hành thử nghiệm trên xe thật tại sa hình chuyên dụng.