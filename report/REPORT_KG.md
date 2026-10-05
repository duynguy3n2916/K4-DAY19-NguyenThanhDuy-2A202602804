# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Thành Duy  **MSSV:** 2A202602804  **Ngày:** 05/10/2026

Số liệu dưới đây lấy từ `ket_qua_benchmark_kg.txt` sau khi chạy `python bench_kg.py --judge` trên mã cuối. Chat: `openai:gpt-4o-mini`; embedding: `openai:text-embedding-3-small`; `top_k=3`, `chunk_size=800`, 176 chunk; graph: 206 node và 384 cạnh. Judge dùng lời gọi LLM riêng, không tính vào chi phí mỗi pipeline.

## 1. Chi phí

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     49.4
graph       196     91958     4864   0.00942    116.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.54
graph       0.94   1.67     2704       88   0.00045     1.86
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0,00112 | 0,00942 | 8,41× |
| Indexing giây | 49,4 | 116,3 | 2,35× |
| Mỗi câu: USD | 0,00013 | 0,00045 | 3,46× |
| Mỗi câu: giây | 1,54 | 1,86 | 1,21× |
| Mỗi câu: in_tok | 694 | 2704 | 3,90× |

Graph tốn thêm 20 lần gọi LLM để trích xuất 20 bài tin khi dựng graph. Ở lúc hỏi, GraphRAG đưa dữ kiện graph và đoạn văn bản vào cùng prompt nên tăng input token và chi phí; cả hai pipeline dùng cùng 176 embedding chunk.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1,00 / 2 | 1,00 / 2 | Hòa | Định nghĩa tiền chất nằm trong một đoạn luật, Flat đã đủ. |
| Q2 | single-hop-news | 1,00 / 2 | 1,00 / 2 | Hòa | Hai tên bị cáo và án tử hình cùng nằm trong một bài báo. |
| Q3 | cross-kb | 0,00 / 0 | 1,00 / 2 | Graph | Graph nối Lê Minh Thành qua vụ và tội danh tới Điều 251, khoản 1. |
| Q4 | cross-kb | 0,00 / 0 | 0,67 / 1 | Graph một phần | Graph lấy được Điều 255 nhưng câu trả lời bỏ sót “tù chung thân” trong khoản 4. |
| Q5 | cross-kb-multi-hop | 0,60 / 1 | 1,00 / 2 | Graph | Graph nối Cái Quang Huy và MDMA tới khoản 4 Điều 250, khung 20 năm/chung thân/tử hình. |
| Q6 | aggregation | 0,00 / 1 | 1,00 / 1 | Graph theo recall | Graph nêu đủ ba tên/đơn vị trong đáp án chuẩn; judge vẫn chấm một phần vì câu trả lời có thêm vụ khác. |

Recall là tỷ lệ chuỗi trong `must_include` xuất hiện nguyên văn. Đây không phải thước đo đầy đủ cho tính đúng pháp lý hoặc mức độ đầy đủ của câu trả lời.

## 3. Phân tích lỗi

### E1 — Cầu nối gãy ở một vụ được trích từ phần tin liên quan

- **Hiện tượng:** Có `Case` về Cái Quang Huy mang `doc_id` của bài Lê Minh Thành nhưng không có cạnh `CHARGED_WITH`, dù phần tin liên quan trong bài gốc có nêu hành vi vận chuyển.
- **Bằng chứng:** Chạy truy vấn sau trên graph sau benchmark:

```cypher
MATCH (k:Case)
WHERE NOT (k)-[:CHARGED_WITH]->()
RETURN k.name AS name, k.doc_id AS doc_id;
```

```text
Vụ vận chuyển ma túy của Cái Quang Huy | news-100260918080821054
Vụ tông cảnh sát giao thông ở An Giang | news-100260926112415229
```

Trong `data/drug_news/news-100260918080821054.md`, đoạn tin liên quan có câu “Cái Quang Huy bị cáo buộc hai lần vận chuyển ma túy về Việt Nam…”, nhưng vụ tương ứng không có tội danh đã liên kết. Vụ An Giang cần xem bài gốc trước khi kết luận có nên nối với luật ma túy hay không.
- **Nguyên nhân:** Corpus chứa cả mẩu tin liên quan bên dưới bài chính; bước LLM trích được vụ phụ nhưng bỏ `charges`. `link_entity` chỉ nối tên tội đã được LLM trả về, không thể sửa một mảng rỗng.
- **Đề xuất sửa:** Ở bước crawl, tách phần tin liên quan khỏi thân bài hoặc lưu thành tài liệu riêng. Trong `extract_news_cases`, đối chiếu các từ khóa tội danh ở từng đoạn tin với danh sách chuẩn khi `charges` rỗng và ghi cờ cần kiểm duyệt. Việc này tăng xử lý và có nguy cơ nối nhầm nếu chỉ dựa vào từ khóa.

### E5 — Câu trả lời lệch dữ kiện graph về mức án tối đa

- **Hiện tượng:** Q4 GraphRAG nói “phạt tù tối đa lên đến 20 năm”, bỏ mức **tù chung thân**. Judge chấm 1/2 dù graph đã có khoản 4 Điều 255.
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt`, Q4 Graph trả lời: “Hành vi này có thể bị phạt tù tối đa lên đến 20 năm theo Điều 255 BLHS … khoản 4.” Kiểm tra khoản trong graph:

```cypher
MATCH (:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause {number:4})
RETURN cl.penalty AS penalty, split(cl.text, '\n')[0] AS first_line;
```

```text
first_line: 4. Phạm tội thuộc một trong các trường hợp sau đây, thì bị phạt tù 20 năm hoặc tù chung thân:
```

- **Nguyên nhân:** Bước sinh câu trả lời rút gọn sai lựa chọn hình phạt từ dữ kiện graph, có thể do prompt chứa nhiều vụ và Điều luật liên quan cùng người “Hoàng Nato”. Đây là lỗi ở bước trả lời, không phải thiếu dữ liệu luật.
- **Đề xuất sửa:** Trong `GRAPH_PROMPT`, yêu cầu liệt kê nguyên văn **mọi lựa chọn** trong khoản có khung cao nhất khi hỏi án tối đa. Có thể chọn vụ theo ngữ cảnh của câu hỏi trước khi đưa vào prompt để giảm nhiễu; đánh đổi là truy vấn xếp hạng vụ phức tạp hơn.

## 4. Kết luận

Flat RAG đủ cho câu hỏi một nguồn Q1–Q2: cả hai đạt recall 1,00 và judge 2. GraphRAG đáng chi phí khi cần nối tin với luật hoặc tổng hợp nhiều vụ: Q3 từ 0/0 lên 1,00/2, Q5 từ 0,60/1 lên 1,00/2; recall trung bình tăng từ 0,43 lên 0,94. Đổi lại indexing tốn 8,41 lần chi phí USD và mỗi câu tốn 3,46 lần. Graph vẫn cần kiểm chứng bước sinh câu trả lời như lỗi Q4.

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

`--check` dựng graph nhỏ (luật + một bài); sau đó `--judge` đã dựng lại graph đầy đủ 206 node / 384 cạnh. Chạy `--judge` xong mới chụp ảnh.

Ảnh Neo4j cần lưu: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`. Người chọn cho ảnh thứ ba: **Cái Quang Huy**. Chụp trực tiếp Neo4j Browser, thấy ô truy vấn và Results overview theo `LAB_GUIDE.md` Bước 8.2.

## Vấn đề gặp phải

Q4 còn thiếu “tù chung thân” dù graph đã có dữ kiện; Q6 có thêm vụ Sầm Sơn ngoài đáp án chuẩn. Hai trường hợp được giữ nguyên trong file benchmark để báo cáo phản ánh đúng lần chạy thật.
