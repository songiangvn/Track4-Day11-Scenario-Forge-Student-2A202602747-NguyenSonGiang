MSSV: 2A202602747

# Lab 11 · Scenario Forge — báo cáo

Họ tên: Nguyễn Sơn Giang.

**Về MSSV:** MSSV của tôi có chữ `A`, nên regex `\d{6,12}` trong `run_all.py` không nhận dòng trên. Mọi bước được chạy bằng biến môi trường mà `run_all.py` đọc trước REPORT.md, cũng là cách grader truyền MSSV:
`LAB11_MSSV=2A202602747 python run_all.py all`. Mọi file evidence đều ghi `_run.mssv = "2A202602747"`, `_run.quick = false`. Tôi không sửa `run_all.py`, `avsim/` hay `tests/`.

Thị trấn của bạn (`meta.json: _run.town`): **A** (p_bin = wet 0,20 · pedLook 0,85 · brakes 0,15; cutGap trung vị 12 m).

Kiểm tra: `python -m pytest -q tests/test_todos.py` cho 31 passed; `tests/test_parity.py` cho 10 passed (có node).

---

## 1. Phát lại log có trung thực không? (bước A)

| `replay.json` | giá trị |
|---|---|
| `n` (kịch bản) | 400 |
| `dropped_human_crash_logs` (log mà chính người lái va chạm, bỏ đi) | **2** → `n_used` = 398 |
| `replay.crashes` (planner trên log phát lại) | **102**, `contact` = **rear 93, side 9** |
| `reactive.crashes` (planner trong thế giới phản ứng) | **0** |
| `crash_in_both` | 0 |

Phần lớn va chạm khi phát lại là `rear` (93/102) và đều là **va chạm giả**. Khi phát lại, xe bám đuôi TG chạy đúng quỹ đạo đã ghi: quỹ đạo đó được sinh ra khi phía trước là *người lái* (HUMAN: T = 1,1 s, b = 3,5). Planner của ta giữ khoảng cách lớn hơn (T = 1,6 s), giảm tốc sớm hơn trước xe chuyển làn và xe buýt, và chạy chậm lại gần vạch bị che (`cautionV` = 7,5). Xe phía sau trong log **không biết** planner phanh sớm hơn người lái, nên nó không phanh theo và lao vào đuôi xe ta. Trong thế giới phản ứng, TG là IDM, thấy xe ta chậm lại thì tự phanh, và 0 vụ xảy ra. Log phát lại vì vậy vừa đếm sai (102 so với 0) vừa đổ lỗi sai (lỗi "bị tông đuôi" nằm ở log, không ở planner).

Ví dụ `replay_only[0]` (scenario 0, chính là run **R-01** trong `viewer.json`; ảnh `evidence/shots/q1_replay_fake_rear_R-01_t17.9.png`). Đường khô, cutGap 14,3 m, tgT 0,87 s. Khi phát lại, xe ta dừng chờ trước vạch có người đi bộ (x ≈ 112 m) và TG đâm vào đuôi lúc 17,9 s; trình xem ghi "va chạm giả (phát lại)". Cùng kịch bản đó trong thế giới phản ứng không có va chạm: `min_ttc_reactive` = 2,63 s, vì TG thấy xe ta dừng và phanh theo.

## 2. Kịch bản bất khả thi (bước B)

Các luật trong `lab11/feasible.py: is_feasible`:

| Luật | Căn cứ vật lý |
|---|---|
| (π/cutDur)²·3,5/2 > μ·g → loại | Một lần chuyển làn dạng cos, rộng 3,5 m, kéo dài cutDur giây, có gia tốc ngang cực đại như vậy. Lốp chỉ cho tối đa μ·g lực bám ngang (vòng tròn ma sát). Đường ướt (μ = 0,5, tức 4,9 m/s²) thì mọi pha chuyển làn ngắn hơn ≈ 1,9 s đều không ai làm được. |
| cutBrake > μ·g → loại | Không xe nào phanh mạnh hơn giới hạn bám của mặt đường. |
| pedV > 2,5 m/s → loại | Trên mức đó là chạy, không còn là đi bộ qua đường: tốc độ đi bộ đo ngoài thực tế khoảng 1,0–1,8 m/s. |
| cutV ∉ (0, 22] m/s → loại | Xe chuyển làn phải đang chạy, và không nhanh quá ~80 km/h trên phố có vạch qua đường và bến xe buýt. |

Theo `hunt.json: infeasible`: bus_bypass 2, cut_in 3, **crosswalk_block 15**, rear_ended 4. Cuộc săn `crosswalk_block` sinh nhiều kịch bản bất khả thi nhất. Phân loại lại `cloud.json` theo luật cho thấy **cả 15 đều do luật gia tốc ngang**. Lý do: điểm nguy hiểm của kiểu này là khoảng phanh còn lại trước người đi bộ, `ped_room = khoảng cách − v²/(2μg)`. Đường ướt làm quãng phanh dài thêm 60 %, nên CEM đẩy `p_bin[wet]` từ 0,20 lên **0,68** (`hunt.json: crosswalk_block.proposal.p_bin`). Trên đường ướt, cùng một cutDur thường gặp lại vượt μ·g. Nói chung, kịch bản "nguy hiểm nhất" thường nằm ở rìa vật lý (đường trơn, chuyển làn gắt, người chạy), nên điểm nguy hiểm kéo q về đúng chỗ đó.

Nếu **đếm** chúng là va chạm, CEM sẽ chọn elite từ những mẫu "va chạm dễ" bất khả thi, q co hẳn vào vùng không tồn tại ngoài đời, và ước lượng IS cộng thêm khối xác suất của các sự kiện không thể xảy ra. Kết quả là xác suất bị thổi phồng, các kiểu thất bại thật bị che, và planner bị "sửa" cho những tình huống không có thật.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: **k = `crashes` = 3**, **n = 3000**, **p̂ = 1,00·10⁻³**, Wilson 95 % **[`lo` 3,40·10⁻⁴, `hi` 2,94·10⁻³]**. Cả 3 vụ đều là `bus_bypass` (`modes`). Ngoài ra có 13 kịch bản bất khả thi bị loại (`infeasible`).

Cần bao nhiêu lần chạy để khoảng hẹp còn ±20 % của p: n ≈ z²(1−p)/(0,2²·p) ≈ **96/p** = 96/10⁻³ ≈ **96 000** lần, gấp 32 lần ngân sách hiện có. Nếu p thật nhỏ như IS gợi ý (≈ 3·10⁻⁴, câu 4) thì cần ≈ 325 000 lần.

Có những kiểu **chưa hề xuất hiện** trong 3 000 lần: `cut_in`, `crosswalk_block`, `rear_ended` và `other` đều có k = 0 (`mc.json: per_mode`). Con số đúng cần báo **không phải 0** mà là cận trên Wilson với k = 0: **P < 1,28·10⁻³** (95 %) cho mỗi kiểu (`per_mode.<kiểu>.hi` = 0,0012789). Khoảng chuẩn (Wald) sẽ cho [0, 0], tức "không bao giờ", và điều đó sai.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng từ `estimate.json: seeds[i].per` (n = 600 mỗi kiểu mỗi seed):

| kiểu | seed | p | hits | ess | top_share | ghi chú |
|---|---|---|---|---|---|---|
| bus_bypass | 0 | 2,09·10⁻⁴ | 64 | 128,9 | 0,357 | |
| bus_bypass | 1 | 5,01·10⁻⁴ | 69 | 107,8 | 0,388 | |
| bus_bypass | 2 | 2,95·10⁻⁴ | 74 | 123,4 | 0,214 | |
| crosswalk_block | 0 | 1,12·10⁻⁷ | 26 | 108,9 | 0,301 | |
| crosswalk_block | 1 | 9,58·10⁻⁸ | 20 | 127,8 | 0,236 | |
| crosswalk_block | 2 | 1,03·10⁻⁷ | 27 | 126,0 | 0,270 | |
| cut_in | 0 / 1 / 2 | **< 0,025** (`upper_zero_hits`) | 0 | 141,2 / 122,8 / 97,9 | – | không thấy: báo cận trên, không báo 0 |
| rear_ended | 0 / 1 / 2 | **< 0,025** (`upper_zero_hits`) | 0 | 117,6 / 120,6 / 106,3 | – | như trên |
| other | (từ `mc.json`) | 0/3000, **< 1,28·10⁻³** | 0 | – | – | `estimate.json: other` |

`total` (các kiểu + other): seed 0 = 2,09·10⁻⁴, seed 1 = 5,01·10⁻⁴, seed 2 = 2,95·10⁻⁴; trung vị **2,95·10⁻⁴**.

**Các seed có đồng ý không?** Với bus_bypass, ba seed nằm trong khoảng ×2,4 của nhau. Với crosswalk_block, ba seed khớp trong ±10 % (≈ 1·10⁻⁷). **So với Monte Carlo:** MC cho bus_bypass 1,0·10⁻³ [3,4·10⁻⁴; 2,9·10⁻³]. Cả ba ước lượng IS đều nằm ngay dưới hoặc tại cận dưới Wilson, tức thấp hơn MC khoảng 2–5 lần. Đó đúng là cái bẫy README cảnh báo: q co hẹp nên ước lượng IS thường **thấp** hơn sự thật 1,5–4 lần. Với crosswalk_block, IS báo ~10⁻⁷ trong khi MC chỉ cho được cận < 1,28·10⁻³. Hai con số không mâu thuẫn, nhưng 10⁻⁷ là xác suất của **chỗ va chạm mà cuộc săn tìm được**: q của nó nằm ở pedV z ≈ −2,2, busGo z ≈ −2,8, đường ướt (`hunt.json: crosswalk_block.proposal`), một góc rất hiếm của p. Không có gì bảo đảm đó là cách chặn lối hay xảy ra nhất (bẫy 3), nên tôi coi 10⁻⁷ là một **chặn dưới thô**, không phải xác suất của kiểu này.

**top_share > 0,5:** trong `estimate.json` không seed nào vượt 0,5 (cao nhất 0,388 ở bus_bypass seed 1). Tuy vậy, cùng loại ước lượng trong `fix.json` có: `baseline_train.crosswalk_block.top_share` = **0,835**, `baseline_test.crosswalk_block` = 0,562, `baseline_test.bus_bypass` = 0,509. Còn `table[0]` (`cautionV 5,0`) có crosswalk_block p = 1,25·10⁻² từ **1 hit**, top_share = **1,00**. Khi top_share lớn, p = mean(w·y) gần như bằng w_max/n: con số chỉ phản ánh trọng số p/q của **một** mẫu. Đổi seed để mẫu đó không được rút trúng thì ước lượng sụt hàng chục lần, và khoảng ±1,96·se cũng hẹp sai vì se tính từ chính vài mẫu đó. Ngược lại, top_share thấp **không** chứng minh ước lượng đúng: nếu q bỏ sót hẳn một vùng va chạm thì không mẫu nào nặng ký cả.

**Ngân sách:** `estimate.json: budget` = **16 800** lần chạy cho cả 4 kiểu, gồm 4 × (8 vòng × 300 săn + 3 seed × 600). Để MC đạt cùng độ chính xác tương đối như IS ở bus_bypass seed 2 (se/p = 9,30·10⁻⁵ / 2,95·10⁻⁴ ≈ 32 %) cần n ≈ (1−p)/(p·0,32²) ≈ **33 000** lần cho riêng kiểu này. Với crosswalk_block ở ~10⁻⁷, MC cần cỡ 10⁹ lần để thấy được vài vụ, trong khi IS dùng 4 200 lần mỗi kiểu (2 400 săn + 1 800 ước lượng). IS rẻ hơn rất nhiều, nhưng chỉ trung thực đến mức đề xuất q phủ đúng vùng va chạm.

**Phần `other`:** không thuộc kiểu nào nên không có điểm nguy hiểm để săn. CEM cần một điểm số liên tục cho từng kiểu, và một kiểu "mọi thứ khác" không có hướng nào để q đi tới. Vì vậy phần này lấy từ Monte Carlo C1 (`estimate.json: other` = {k 0, n 3000, hi 1,28·10⁻³, source mc.json}). Tổng = các kiểu + other.

## 5. Các kiểu thất bại (bước C2 + 3D)

`hunt.json: found` (số va chạm theo nhãn mà mỗi cuộc săn tìm được):

| cuộc săn | found |
|---|---|
| bus_bypass | {bus_bypass: **109**} |
| cut_in | {bus_bypass: 17} (0 cut_in) |
| crosswalk_block | {bus_bypass: 2, crosswalk_block: **3**} |
| rear_ended | {bus_bypass: 15, crosswalk_block: 6} (0 rear_ended) |

**bus_bypass** (ảnh `evidence/shots/q5_bus_bypass_B-01_t12.4.png`, run B-01, đường ướt, cutGap 4,1 m). Xe buýt đỗ ở làn ngoài E2. Xe tím C2 bị kẹt sau xe buýt nên lách sang làn E1 của xe ta, chấp nhận một khe chỉ 4,1 m. Lúc lách, C2 vẫn đang chậm (< 6 m/s) và ở x < 112 m. Planner chỉ coi C2 là "xe phía trước" khi độ lệch ngang < `cutinLat` = 2,15 m. Trong lúc C2 mới ló đầu vào làn, planner chưa coi nó là chướng ngại nên vẫn giữ tốc độ, đến khi nhận ra thì trên đường ướt không phanh kịp, và va chạm xảy ra lúc 12,4 s.

**crosswalk_block** (ảnh `evidence/shots/q5_crosswalk_block_C-01_t20.8.png`, run C-01, đường ướt). Người đi bộ P1 không ngó qua xe buýt (pedLook = 0) nên không thấy xe ta, và vì xe buýt che, xe ta cũng không thấy họ. Họ bắt đầu băng qua vạch. Khi P1 hiện ra sau đầu xe buýt, xe ta phanh gấp, nhưng trên đường ướt quãng phanh dài nên xe dừng với thân xe nằm trên vạch qua đường. P1 phải đứng lại trước thân xe: một sự kiện `block` lúc 20,8 s.

**Kiểu không tìm thấy:** `cut_in` và `rear_ended`. Cuộc săn của chính chúng không ra hit nào đúng kiểu, IS báo cận < 0,025 mỗi seed, và MC báo cận < 1,28·10⁻³. Tôi tin ở mức vừa phải rằng chúng hiếm hơn bus_bypass. Planner giữ T = 1,6 s nên cắt làn ở chỗ thoáng ít khi thành va chạm, và TG là IDM phản ứng nên hiếm khi tông đuôi. Nhưng "không tìm thấy" chỉ cho một cận trên, không phải 0: điểm nguy hiểm của các cuộc săn này bị bus_bypass "hút" mất (cut_in hunt ra 17 bus_bypass, rear_ended hunt ra 15 bus_bypass và 6 crosswalk_block).

**Nếu chỉ chạy một cuộc săn chung:** CEM co về kiểu dễ va chạm nhất. Cuộc săn bus_bypass ra 109 vụ, và cả cuộc săn nhắm vào cut_in lẫn rear_ended cũng trôi về bus_bypass. Một cuộc săn chung sẽ gần như chỉ thấy bus_bypass, và crosswalk_block (chỉ 3 vụ dù có cuộc săn riêng) sẽ bị bỏ sót hoàn toàn. Vì vậy mỗi kiểu cần một cuộc săn với điểm nguy hiểm riêng.

## 6. Sửa planner (bước E + 3D)

Chọn trên TRAIN bằng `choose_policy` (12 ứng viên gốc trong `policy_candidates.json`, tôi không thêm ứng viên). `fix.json`:

| | policy | fail | fail_hi_seen | fail_hi | unseen | progress (m) | harsh (khung/lần chạy) |
|---|---|---|---|---|---|---|---|
| `baseline_train` (TRAIN) | gốc | 4,13·10⁻⁴ | 8,25·10⁻⁴ | 8,25·10⁻⁴ | [] | 214,8 | 7,60 |
| `chosen` (TRAIN) | **{cutinLat: 3,0}** | 1,04·10⁻⁷ | 2,76·10⁻⁷ | 2,76·10⁻⁷ | [bus_bypass] | 214,4 | 7,58 |
| `chosen_test` (TEST) | {cutinLat: 3,0} | **1,05·10⁻⁶** | **2,47·10⁻⁶** | **3,88·10⁻²** | [bus_bypass] | 215,6 | 7,73 |
| `baseline_test` (TEST) | gốc | **3,78·10⁻⁴** | 7,81·10⁻⁴ | **2,06·10⁻³** | [] | 215,7 | 8,18 |

TRAIN chỉ để sàng, con số tôi báo là TEST: seed mới, và một cuộc săn mới nhắm vào chính policy đó.

**Vì sao `fail_hi` của bản sửa (3,88·10⁻²) lớn hơn cả `fail` của policy gốc (3,78·10⁻⁴), dù bản sửa có thể tốt hơn thật?** Trên TEST, cuộc săn bus_bypass nhắm lại vào `cutinLat 3,0` cho **0 hit** (`chosen_test.bus_bypass.hits` = 0). Một kiểu không có hit nào chỉ được phép báo bằng cận `upper_zero_hits` = 3/(0,2·400) = **0,0375**, và con số đó được cộng vào `fail_hi`. Thêm cận của va chạm khác trên 3 000 lần tự nhiên (`other_crashes.hi` = 1,28·10⁻³) và crosswalk_block (2,47·10⁻⁶) thì ra 3,88·10⁻². Đó là giới hạn của ngân sách (n = 400 không chứng nhận được gì dưới 10⁻²), không phải bằng chứng bản sửa tệ hơn. Cái dữ liệu thực sự cho thấy là `fail_hi_seen` = 2,47·10⁻⁶ so với 7,81·10⁻⁴: bản sửa xoá bus_bypass trên mọi mẫu đã thử. Quãng đường gần như không đổi (215,6 so với 215,7 m), và số khung phanh gấp giảm (7,73 so với 8,18).

Cơ chế của bản sửa: với `cutinLat` = 3,0 m, planner coi C2 là "xe phía trước" ngay khi C2 mới bắt đầu lấn sang để lách xe buýt, nên giảm tốc sớm và chừa khe cho C2. Ảnh `evidence/shots/q6_ghost_fix_avoids_B-04_t12.5.png` (run B-04): xe đặc là policy gốc, va chạm C2 lúc 12,5 s. **Bóng mờ** là bản sửa `cutinLat 3,0` trên cùng kịch bản (trình xem đặt bóng mờ là bản sửa cho những run được tìm dưới policy gốc): nó ở lùi phía sau và không va chạm. Ô "chính sách gốc: không va chạm" trên ảnh là nhãn của bóng mờ.

**Núm "nghe thì an toàn hơn" nhưng làm tăng thất bại:** `cautionV` = 5,0, tức đi chậm hơn nữa gần vạch bị xe buýt che. Theo `fix.json: table[0]`, TRAIN fail = **1,25·10⁻²** (hi 3,70·10⁻²), gấp ~30 lần policy gốc (4,13·10⁻⁴), phanh gấp 12,1 khung/lần chạy (gốc 7,6), quãng đường 173,4 m (−19 %). Thế giới phản ứng lại như sau. Người đi bộ P1 quyết định bước ra dựa trên **thời gian** xe tới nơi, tta = khoảng cách / tốc độ, so với `pedGap` (avsim/core.py, khối `walk`). Xe ta bò chậm thì tta dài ra, nên người đi bộ thấy "còn đủ thời gian" và bước ra **khi xe ta đã ở gần hơn về khoảng cách**. Xe ta phải phanh gấp và dễ dừng với thân trên vạch: crosswalk_block tăng. Lưu ý trung thực: con số 1,25·10⁻² này dựa trên **1 hit** với top_share = 1,00, nên độ lớn không đáng tin. Nhưng hướng thay đổi nhất quán với một ví dụ thứ hai: `T` = 2,0 (giữ khoảng cách xa hơn) làm crosswalk_block tăng từ 10 hit lên **28 hit** và p từ 1,04·10⁻⁷ lên 3,59·10⁻⁷ (`table[8]`). Cơ chế giống trên: xe ta giữ khoảng cách xa hơn nên đến vạch muộn hơn và tta dài hơn trong mắt người đi bộ. Không phải núm "an toàn hơn" nào cũng an toàn hơn.

## 7. Từ mô phỏng ra đường thật

| Giả định trong `avsim/space.py` | Dữ liệu thật thay nó | Nếu sai thì con số sai theo hướng nào |
|---|---|---|
| **Khe chấp nhận khi chuyển làn** `cutGap` ~ logN(trung vị 12 m, σ 0,5), `cutDur` ~ logN(4 s, 0,35) | Quỹ đạo chuyển làn quay từ drone: highD / exiD (Krajewski et al., 2018/2022), NGSIM US-101; tốt nhất là video tại chính các bến xe buýt (xe lách xe buýt đỗ) | bus_bypass gần như chỉ xảy ra ở đuôi khe nhỏ (`hunt.json`: q của bus_bypass ở cutGap z ≈ −1,9). Nếu tài xế thật ở bến xe buýt chấp nhận khe nhỏ hơn (đô thị đông, xe máy) thì P(bus_bypass) **bị đánh giá thấp**, có thể nhiều lần, vì nó nhạy với đuôi phân phối. |
| **Người đi bộ ngó qua xe buýt** `pedLook` = 0,85 và **khe chấp nhận** `pedGap` ~ logN(4,5 s) | Video quan sát tại vạch qua đường cạnh bến xe buýt; bộ dữ liệu người đi bộ ở nút giao như inD (Bock et al., 2020); nghiên cứu gap acceptance của người đi bộ | crosswalk_block cần pedLook = 0 (CEM đẩy p_bin[pedLook] về 0,03). Nếu ít người ngó hơn 85 % hoặc họ chấp nhận khe ngắn hơn, crosswalk_block và rắc rối câu 9 **bị đánh giá thấp**. |
| **Ma sát và tần suất đường ướt** (wet = 0,20, μ = 0,5 so với 0,8, chỉ hai mức) | Số giờ mưa/đường ướt từ trạm khí tượng địa phương, cộng ma sát đo bằng skid trailer (ASTM E274) trên chính mặt đường đó | Cả 3 vụ MC và hầu hết q của các cuộc săn đều nghiêng về đường ướt. Nếu khí hậu mưa nhiều hơn (như thị trấn B) hoặc μ ướt thật < 0,5 (mặt đường mòn, nước đọng) thì xác suất **bị đánh giá thấp**. Ngược lại, μ thật thay đổi liên tục chứ không phải hai mức, nên luật khả thi cũng xê dịch. |

## 8. Giới hạn

1. **Một con phố, một bến xe buýt, một kịch bản 24 s**, và mỗi lần chạy được coi là một "chuyến". Mọi xác suất là *mỗi lần đi qua đúng cảnh này*, không phải mỗi km hay mỗi giờ lái. Không có giao lộ khác, không có xe máy, không có đêm hay mưa làm giảm tầm nhìn.
2. **Người lái khác là IDM có lý trí**: gap acceptance cố định, không mất tập trung, không vượt đèn, phản ứng tức thì không trễ. Người đi bộ không bao giờ bước vào thân xe. Va chạm do lỗi con người thật (nhìn điện thoại, say rượu) không có trong p(u).
3. **Cảm biến hoàn hảo trừ phần bị xe buýt che**: không nhiễu, không trễ, không nhận nhầm, và chỉ xe buýt che tầm nhìn. Xe ta chỉ phanh, không đánh lái tránh. Phân phối tự nhiên là giả định dạy học, không khớp dữ liệu (câu 7), nên mọi con số là có điều kiện trên giả định đó.

Nếu có thêm một tuần: (a) đo độ nhạy của P(bus_bypass) với `cutGap` và `wet` bằng cách quét các thị trấn A–D cùng một pipeline; (b) thêm nhiễu và trễ nhận thức (100–300 ms) rồi săn lại; (c) chạy MC lớn (~100 000 lần) để kiểm tra độ lệch thấp của IS ở bus_bypass; (d) săn có điều kiện cho rắc rối ở câu 9 cùng với logic sửa đề xuất.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** **người băng ngang ngay sau đuôi xe buýt đỗ** (hazard `ped`, lề nam S, x = 95 m, v = 1,6 m/s, bước ra khi xe ta còn cách 34 m). Ngoài đời: hành khách vừa xuống xe buýt, không đi tới vạch (x = 122 m) mà băng tắt ngay sau đuôi xe buýt (xe buýt chiếm x ≈ 100–112 m). Ở cảnh dựng, người đó bước ra ở x = 95 m, giữa đuôi xe buýt và xe C2 đang xếp hàng sau xe buýt ở làn ngoài, rồi đi ngang qua làn ngoài trước mũi C2 để sang làn xe ta. Đây là kiểu "dart-out giữa khối nhà" kinh điển.
- **Căn cứ:** đọc `avsim/core.py`, planner chỉ dự đoán người đi bộ **P1 trên vạch** (`pedBuffer`, `t_enter < t_arrive + pedBuffer`). Người băng ngang ngoài vạch chỉ được coi là chướng ngại khi **đã vào hành lang** của xe ta (|lat| ≤ 1,9 m), và lúc đó planner phanh bằng IDM. `cautionV` chỉ có tác dụng khi *P1* bị che. Vì vậy tôi đoán planner sẽ phản ứng muộn.
- **Giả định (`hazard.json: why`):** `space` = {h0_x [92, 112] m, h0_v [1,0, 2,2] m/s, h0_trig [10, 40] m}, `exposure` = 0,02/chuyến.
  - x là vùng xe buýt chiếm (x ≈ 100–112 m) và sát đuôi nó.
  - v từ số đo tốc độ đi bộ của Knoblauch, Pietrucha & Nitzburg 1996 (≈ 1,0 m/s người già đến ~1,5 m/s trung bình), cộng mức đi vội 2,2 m/s, vẫn dưới ngưỡng chạy 2,5 m/s của `is_feasible`.
  - trig 10–40 m tương ứng ~1–4 s ở 8–12 m/s, ngắn hơn pedGap trung vị 4,5 s của thị trấn vì chính xe buýt che tầm nhìn của người đó.
  - exposure: mỗi chuyến đi qua đúng một bến có xe buýt đỗ. Dart-out giữa khối nhà là kiểu tai nạn người đi bộ phổ biến trong phân loại của Snyder & Knoblauch 1971 và Hunter et al. 1996 (FHWA-RD-95-163). Còn **1 lần trong 50 lần qua bến là giả định bậc độ lớn của tôi**, chưa có số đo; cần đếm thực địa để thay, và `per_trip` tỉ lệ thuận với nó.
- **Kết quả** (`mine.json`; 1 495/1 500 phiên bản khả thi, **`accepted` = 0,997**; `exact.planner` = [ego, H0, 12,9 s], tức đúng cảnh tôi dựng cũng va chạm khi chạy lại):

| `mine.json` | baseline (gốc) | chosen ({cutinLat 3,0}) |
|---|---|---|
| n | 1 495 | 1 495 |
| crashes | 33 | 32 |
| **avoidable** (phanh hết cỡ từ lúc thấy thì tránh được) | **32** | **32** |
| unavoidable | 0 | 0 |
| unattributed (va chạm thường của thị trấn) | 1 | 0 |
| low_speed (< 1 m/s) | 3 | 3 |
| avoidable_foreknown | 32 | 32 |
| **P(lỗi planner \| rắc rối)** = avoidable/n, Wilson 95 % | **2,14·10⁻² [1,52·10⁻², 3,01·10⁻²]** | **2,14·10⁻² [1,52·10⁻², 3,01·10⁻²]** |
| `per_trip` = exposure × p (cận trên) | **4,28·10⁻⁴** (6,01·10⁻⁴) | 4,28·10⁻⁴ (6,01·10⁻⁴) |

  - **So với lách xe buýt:** tần suất bus_bypass mỗi chuyến là 2,95·10⁻⁴ (`estimate.json`, trung vị `total`, gần như toàn bộ là bus_bypass) hoặc 1,0·10⁻³ (`mc.json`). Rắc rối của tôi ở **cùng bậc độ lớn** (4,3·10⁻⁴), tức một kiểu thất bại mà các cuộc săn ở bước C không thể thấy, vì thế giới gốc không có người đi bộ ngoài vạch. Con số này tỉ lệ thuận với exposure 0,02 mà tôi giả định.
  - **Phân loại:** 32/33 va chạm là **avoidable**: phanh hết cỡ ngay từ khung đầu tiên người đó vừa bước ra *và* cảm biến thấy được thì xe ta đã dừng kịp. Vậy đây là **lỗi planner**, không phải tai nạn bất khả kháng (0 unavoidable). Vụ **unattributed** duy nhất xảy ra khi rắc rối chưa bắt đầu (một va chạm thường của thị trấn), nên không nói gì về rắc rối này. 3 vụ `low_speed` xảy ra khi xe ta đã gần đứng yên (< 1 m/s), vẫn được đếm trong loại của chúng. `avoidable_foreknown` = `avoidable` = 32: ở đây cái giá của việc bị che là 0, nghĩa là dù chỉ phản ứng từ lúc *thấy* được thì vẫn kịp, và lỗi hoàn toàn ở chỗ planner phản ứng chậm sau khi đã thấy.
  - **Bản sửa ở bước E không cứu:** `cutinLat 3,0` chỉ đổi cách planner xử lý C2, nên vẫn 32 avoidable, cùng p.
  - **"Tránh được" là chuẩn lạc quan theo hai hướng.** (i) Người phanh trong phép thử được *biết trước* rắc rối ngay khung đầu tiên nó nhìn thấy được, và phanh tối đa μg không trễ. Một planner thật cần thời gian để phát hiện, xác nhận và quyết định, nên `avoidable` **đếm nhiều hơn** số vụ một planner thật có thể tránh. Nó là cận trên của "lỗi planner". (ii) Phép thử chỉ thử **phanh**, không thử đánh lái hay kết hợp. Một số vụ bị xếp `unavoidable` có thể tránh được bằng cách lách, nên theo hướng này `avoidable` lại **đếm ít đi**. Ở đây unavoidable = 0 nên hướng (ii) không đổi kết quả, còn hướng (i) có nghĩa 2,14·10⁻² là cận trên của tỉ lệ lỗi mà một planner thực tế có thể sửa được.
- **Hình:** `evidence/shots/q9_hazard_ped_behind_bus_t12.9.png` là chương 10 với `hazard.json` nạp vào, đúng lúc va chạm 12,9 s (H0 "người băng ngang" chạm đầu xe ta ở x ≈ 94 m, giữa đuôi xe buýt và C2 đang xếp hàng ở làn ngoài). `evidence/shots/q9_hazard_ch9_GB-01_t12.85.png` là chương 9, run GB-01 (một phiên bản của `mine` trong giao thông thị trấn, đường khô), va chạm lúc 12,85 s; bản sửa (bóng mờ) cũng va chạm lúc 12,9 s. Ảnh `..._t11.0.png` là 1,9 s trước đó: người đi bộ đã ở giữa làn ngoài, trước mũi C2, **nhìn thấy rõ** từ xe ta, nhưng vì họ chưa vào hành lang |lat| ≤ 1,9 m của xe ta nên planner chưa coi họ là chướng ngại. Ảnh này khớp với `avoidable_foreknown` = `avoidable`: lỗi không nằm ở tầm nhìn bị che mà ở chỗ planner không dự đoán người đang đi về phía làn mình. Trình xem JS cho cùng thời điểm va chạm với Python (12,9 s).
- **Bạn tự lái thì sao:** tôi **không** tự lái (ảnh được chụp bằng trình duyệt tự động, không có thao tác bàn đạp), nên không có số "you" trong `mine.json: exact`. Thay cho người lái, phép thử "phanh hết cỡ ngay khi thấy" ở trên cho thấy phản ứng khác planner là **phanh mạnh ngay khi người vừa ló ra khỏi đuôi xe buýt**, thay vì chờ họ vào hẳn làn.
- **Cách sửa:** tôi quét thêm vài núm trên **đúng 1 495 phiên bản của `mine`**. Đây là lần chạy thêm ngoài evidence/ (script trong scratchpad, cùng seed `test + 404`), và progress ở đây đo trên 600 lần tự nhiên riêng nên khác `fix.json`:

  | policy (chạy thêm) | avoidable | progress (m) |
  |---|---|---|
  | gốc | 32 | 211,3 |
  | pedBuffer 6,0 | 32 | 211,3 |
  | b 4,0 | 24 | 210,7 |
  | T 2,5 | 22 | 190,2 |
  | cautionV 5,0 | 7 | 171,3 |
  | cautionV 4,0 + b 4,0 | 4 | 163,6 |

  `pedBuffer` vô dụng vì nó chỉ áp cho P1. `cautionV` thấp sửa được phần lớn, nhưng mất ~19–23 % quãng đường (vượt luật 5 %), và câu 6 cho thấy nó làm crosswalk_block tăng. `b` = 4,0 (cho phép phanh mạnh hơn) giúp được một phần (−25 %) mà gần như không mất quãng đường, nên là núm đáng thử nhất trong 9 núm, nhưng không đủ. Không núm nào sửa trọn, nên planner cần **logic mới**: (1) **dự đoán mọi người đi bộ**, không chỉ P1 trên vạch: một người đang đi về phía làn xe ta (vận tốc ngang hướng vào làn) được xử lý như P1, với t_enter so với t_arrive + pedBuffer, trước khi họ bước vào hành lang; (2) **thận trọng theo vùng bị che bởi bất kỳ xe đỗ cao nào** (xe buýt, xe tải), không chỉ quanh vạch: khi đi ngang một vùng bị che, giới hạn tốc độ sao cho quãng phanh ở μ hiện tại nhỏ hơn khoảng cách tới mép vùng che. Cách này chỉ làm chậm trong đoạn ~12 m cạnh xe buýt chứ không chậm cả 55 m như `cautionV`, nên mất ít quãng đường.
