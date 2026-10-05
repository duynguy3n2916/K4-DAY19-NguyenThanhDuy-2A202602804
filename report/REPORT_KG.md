# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Thành Duy  **MSSV:** 2A202602804  **Ngày:** 05/10/2026

Số liệu dưới đây lấy từ `ket_qua_benchmark_kg.txt` sau khi chạy `python bench_kg.py --judge` trên mã cuối. Chat: `openai:gpt-4o-mini`; embedding: `openai:text-embedding-3-small`; `top_k=3`, `chunk_size=800`, 176 chunk; graph: 206 node và 385 cạnh. Judge dùng lời gọi LLM riêng, không tính vào chi phí mỗi pipeline.

## 1. Chi phí

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    136.9
graph       196     91958     4909   0.00945    213.0

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     2.50
graph       0.94   1.83     2740       95   0.00046     3.24
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0,00112 | 0,00945 | 8,44× |
| Indexing giây | 136,9 | 213,0 | 1,56× |
| Mỗi câu: USD | 0,00013 | 0,00046 | 3,54× |
| Mỗi câu: giây | 2,50 | 3,24 | 1,30× |
| Mỗi câu: in_tok | 694 | 2740 | 3,95× |

Graph tốn thêm 20 lần gọi LLM để trích xuất 20 bài tin khi dựng graph. Ở lúc hỏi, GraphRAG đưa dữ kiện graph và đoạn văn bản vào cùng prompt nên tăng input token và chi phí; cả hai pipeline dùng cùng 176 embedding chunk.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1,00 / 2 | 1,00 / 2 | Hòa | Định nghĩa tiền chất nằm trong một đoạn luật, Flat đã đủ. |
| Q2 | single-hop-news | 1,00 / 2 | 1,00 / 2 | Hòa | Hai tên bị cáo và án tử hình cùng nằm trong một bài báo. |
| Q3 | cross-kb | 0,00 / 0 | 1,00 / 2 | Graph | Graph nối Lê Minh Thành qua vụ và tội danh tới Điều 251, khoản 1. |
| Q4 | cross-kb | 0,00 / 0 | 1,00 / 2 | Graph | Graph nối hành vi với Điều 255 và nêu đủ mức tối đa 20 năm hoặc tù chung thân. |
| Q5 | cross-kb-multi-hop | 0,60 / 1 | 1,00 / 2 | Graph | Graph nối Cái Quang Huy và MDMA tới khoản 4 Điều 250, khung 20 năm/chung thân/tử hình. |
| Q6 | aggregation | 0,00 / 1 | 0,67 / 1 | Graph theo recall | Graph nêu Cái Quang Huy và Lê Minh Thành nhưng bỏ Viện Pháp y tâm thần, đồng thời thêm vụ 36kg. |

Recall là tỷ lệ chuỗi trong `must_include` xuất hiện nguyên văn. Đây không phải thước đo đầy đủ cho tính đúng pháp lý hoặc mức độ đầy đủ của câu trả lời.

## 3. Phân tích lỗi

### E4 — Recall theo chuỗi không phản ánh hết câu trả lời

- **Hiện tượng:** Q6 Flat có recall `0,00` nhưng judge chấm `1/2`, tức vẫn có phần nội dung liên quan.
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt`, Q6 Flat nêu “Vụ việc của Thành liên quan đến 5 viên nén màu trắng được xác định là ma túy MDMA” và “Vụ việc của Đông liên quan đến 0,686g ma túy MDMA”. Ba chuỗi `must_include` của Q6 trong `data/benchmark_kg.json` là `Cái Quang Huy`, `Lê Minh Thành`, `Pháp y tâm thần`; không chuỗi nào xuất hiện nguyên văn trong câu trả lời nên recall bằng 0, dù judge nhận ra nội dung đúng một phần.
- **Nguyên nhân:** `keyword_recall` trong `bench_kg.py` chỉ tìm chuỗi con chính xác. Tên rút gọn như “Thành” hoặc mô tả vụ việc không được tính; ngược lại việc chỉ nhắc đúng tên cũng chưa chứng minh câu trả lời đúng.
- **Đề xuất sửa:** Bổ sung tập alias được kiểm duyệt cho từng thực thể và chấm cả quan hệ người–vụ–chất, giữ judge hoặc kiểm thủ công cho trường hợp mâu thuẫn. Cách này tốn công tạo nhãn và có thể phát sinh nhận nhầm alias.

### E5 — Câu tổng hợp lệch dữ kiện trong graph

- **Hiện tượng:** Q6 GraphRAG kể thêm vụ mua bán hơn 36kg tại TP.HCM như một vụ có MDMA nhưng bỏ vụ Viện Pháp y tâm thần Trung ương. Judge chấm `1/2` dù recall theo từ khóa đạt `0,67`.
- **Bằng chứng:** Q6 Graph trong file benchmark nói về vụ 36kg: “MDMA cũng được đề cập đến”. Truy vấn trực tiếp trên graph sau benchmark:

```cypher
MATCH (k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})
RETURN k.name AS case_name, k.doc_id AS doc_id ORDER BY case_name;
```

```text
Vụ góp tiền mua ma túy tại Hà Nội | news-100260918080821054
Vụ tổ chức sử dụng ma túy tại Sầm Sơn | news-100260930085028036
Vụ vận chuyển ma túy của Cái Quang Huy | news-100260918080821054
Vụ vận chuyển ma túy từ Đức về Việt Nam | news-100260917203001265
Vụ án tại Viện Pháp y tâm thần Trung ương | news-100260924105118645
```

Danh sách graph không có vụ hơn 36kg nhưng có vụ Viện Pháp y; graph còn tách vụ Cái Quang Huy thành hai `Case` do một bài chứa phần tin liên quan.
- **Nguyên nhân:** Câu trả lời được LLM sinh từ cả chunk vector và dữ kiện graph; prompt chưa buộc kiểm chứng từng vụ theo cạnh `INVOLVES`. Phần tin liên quan trong corpus cũng tạo vụ trùng và làm ngữ cảnh nhiễu.
- **Đề xuất sửa:** Với câu hỏi liệt kê theo chất, lấy danh sách `Case` trực tiếp bằng Cypher rồi yêu cầu LLM chỉ diễn đạt các hàng được trả về; gộp vụ trùng theo thực thể và nguồn trước khi trả lời. Đổi lại cần quy tắc nhận diện câu tổng hợp và gộp vụ đáng tin cậy.

## 4. Kết luận

Flat RAG đủ cho câu hỏi một nguồn Q1–Q2: cả hai đạt recall 1,00 và judge 2. GraphRAG đáng chi phí khi cần nối tin với luật: Q3 từ 0/0 lên 1,00/2, Q4 từ 0/0 lên 1,00/2 và Q5 từ 0,60/1 lên 1,00/2; recall trung bình tăng từ 0,43 lên 0,94. Đổi lại indexing tốn 8,44 lần chi phí USD và mỗi câu tốn 3,54 lần. Câu tổng hợp Q6 cho thấy vẫn phải kiểm chứng danh sách vụ do LLM sinh.

## 5. Tự kiểm

```text
$ python -m pytest tests -q
................................................                         [100%]
48 passed in 0.10s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 292 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00076.
```

`--check` dựng graph nhỏ (luật + một bài); sau đó `--judge` đã dựng lại graph đầy đủ 206 node / 385 cạnh. Hai ảnh đường đi được chụp sau lần dựng graph đầy đủ này; ảnh đếm node vẫn khớp vì số node theo từng label không đổi.

Ảnh Neo4j cần lưu: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`. Người chọn cho ảnh thứ ba: **Cái Quang Huy**. Chụp trực tiếp Neo4j Browser, thấy ô truy vấn và Results overview theo `LAB_GUIDE.md` Bước 8.2.

## Vấn đề gặp phải

Q6 còn thêm vụ hơn 36kg không có cạnh MDMA trong graph và bỏ vụ Viện Pháp y tâm thần. Câu trả lời và điểm số được giữ nguyên trong file benchmark để báo cáo phản ánh đúng lần chạy thật.
