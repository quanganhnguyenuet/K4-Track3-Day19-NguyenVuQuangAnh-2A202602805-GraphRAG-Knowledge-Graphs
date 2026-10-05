# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Vũ Quang Anh  **MSSV:** 2A202602805  **Ngày:** 5/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     48.3
graph       196     91958     4936   0.00947    113.8

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     1.48
graph       0.74   1.50     3058       98   0.00051     1.85
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00947 | ×8.46 |
| Indexing giây | 48.3 | 113.8 | ×2.36 |
| Mỗi câu: USD | $0.00013 | $0.00051 | ×3.92 |
| Mỗi câu: giây | 1.48 | 1.85 | ×1.25 |
| Mỗi câu: in_tok | 694 | 3058 | ×4.41 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> GraphRAG tốn thêm ở giai đoạn indexing vì ngoài 176 lần embedding giống Flat RAG, hệ thống gọi LLM để trích xuất các vụ việc và tạo graph (196 calls, 4.936 output tokens). Khi truy vấn, graph facts được thêm vào prompt nên số input token tăng 4.41 lần; chi phí mỗi câu tăng 3.92 lần nhưng độ trễ chỉ tăng từ 1.48 giây lên 1.85 giây.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai đều truy xuất trực tiếp đúng định nghĩa tiền chất từ KB luật. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Vector retrieval đã lấy đúng bài báo; graph chỉ bổ sung Điều 251 nhưng không cải thiện điểm. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph nối người/vụ án với Crime và Điều 251, nên có cả mức án, tội danh và khung cơ bản. |
| Q4 | cross-kb | 0.00 / 0 | 0.33 / 1 | Graph | Graph tìm đúng hành vi của Hoàng Nato nhưng KG3 thiếu khoản 4 nên chưa trả lời được mức tối đa. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Graph | Graph bổ sung khoản 4 và mức phạt, nhưng LLM vẫn nhầm Điều 250 thành Điều 251. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph | Quan hệ `INVOLVES` giúp GraphRAG nêu thêm các vụ MDMA, dù vẫn thiếu một vụ trong đáp án chuẩn. |

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E2: Thiếu ngữ cảnh luật — bỏ sót khung hình phạt tối đa

- **Hiện tượng:** Ở Q4, GraphRAG đã xác định đúng hành vi của Hoàng Nato là “tổ chức sử dụng trái phép chất ma túy”, nhưng lại trả lời: “Tuy nhiên, trong ngữ cảnh hiện tại không đủ thông tin để xác định mức phạt tù tối đa…”. Kết quả này bỏ sót phần “tù 20 năm hoặc tù chung thân”.
- **Bằng chứng:** `ket_qua_benchmark_kg.txt` ghi Q4 GraphRAG có `recall=0.33`, `judge=1` và nguyên văn câu trả lời nêu trên. Trong khi đó, graph có sẵn Điều 255 khoản 4 cho các vụ chứa “Hoàng Nato”:

```cypher
MATCH (k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)-[:HAS_CLAUSE]->(cl:Clause)
WHERE k.name CONTAINS 'Hoàng Nato'
  AND c.name = 'tổ chức sử dụng trái phép chất ma túy'
  AND cl.number = 4
RETURN DISTINCT k.name AS case_name, a.id AS article, cl.number AS clause, cl.penalty AS penalty
ORDER BY case_name;
```

```
case_name: Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy
article: Điều 255 BLHS | clause: 4 | penalty: phạt tù 20 năm hoặc tù chung thân

case_name: Vụ bắt giữ TikToker Phannhibeauty và giang hồ 'Hoàng Nato'
article: Điều 255 BLHS | clause: 4 | penalty: phạt tù 20 năm hoặc tù chung thân

case_name: Vụ sử dụng ma túy etomidate của Hoàng Nato và Phan Kim Nhi
article: Điều 255 BLHS | clause: 4 | penalty: phạt tù 20 năm hoặc tù chung thân
```

- **Nguyên nhân:** Lỗi nằm ở KG3 (`Neo4jGraph.context`). Quy tắc hiện tại chỉ đưa khoản 1 và các khoản nhắc đúng `Substance` của vụ án. Khoản 4 Điều 255 là khung cao nhất nhưng không được chọn từ truy vấn này, nên prompt gửi LLM không có dữ kiện cần thiết. Đây là đánh đổi giữa prompt ngắn và độ đầy đủ của dữ kiện luật.
- **Đề xuất sửa:** Trong `src/graph.py`, khi câu hỏi có các từ như “tối đa”, “cao nhất”, “khung hình phạt”, bổ sung khoản có `number` lớn nhất của Article đang truy xuất (hoặc luôn thêm khoản cao nhất bên cạnh khoản 1). Cách này tăng token/prompt và có thể đưa thêm ngữ cảnh không liên quan, nhưng giúp câu hỏi về mức phạt tối đa không bị thiếu dữ kiện.

### Lỗi E5: LLM lệch với dữ kiện trong graph

- **Hiện tượng:** Ở Q5, GraphRAG trả lời Cái Quang Huy áp dụng “khoản 4 của Điều 251 BLHS”. Điều luật này không khớp với quan hệ đã lưu trong knowledge graph; tội vận chuyển phải nối tới Điều 250 BLHS.
- **Bằng chứng:** `ket_qua_benchmark_kg.txt` ghi nguyên văn câu trả lời GraphRAG: “khoản áp dụng tương ứng là khoản 4 của Điều 251 BLHS…”. Truy vấn trực tiếp graph cho Cái Quang Huy trả về Điều 250:

```cypher
MATCH (p:Person {name:'Cái Quang Huy'})-[:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)-[:HAS_CLAUSE]->(cl:Clause)
WHERE cl.number = 4
RETURN p.name AS person, k.name AS case_name, c.name AS crime,
       a.id AS article, cl.number AS clause, cl.penalty AS penalty;
```

```
person: Cái Quang Huy
case_name: Vụ vận chuyển ma túy của Cái Quang Huy
crime: vận chuyển trái phép chất ma túy
article: Điều 250 BLHS | clause: 4
penalty: phạt tù 20 năm, tù chung thân hoặc tử hình

person: Cái Quang Huy
case_name: Vụ vận chuyển ma túy từ Đức về Việt Nam
crime: vận chuyển trái phép chất ma túy
article: Điều 250 BLHS | clause: 4
penalty: phạt tù 20 năm, tù chung thân hoặc tử hình
```

- **Nguyên nhân:** Lỗi nằm ở bước sinh câu trả lời của LLM (KG4), không phải ở quan hệ trong graph. Prompt hiện chỉ yêu cầu trả lời dựa trên ngữ cảnh, không buộc model sao chép chính xác `article_id` từ graph. Các chunk vector hoặc nhiều điều luật xuất hiện đồng thời khiến model nhầm Điều 250 với Điều 251, dù graph fact đã có dữ kiện đúng.
- **Đề xuất sửa:** Đưa các graph facts quan trọng vào một phần có cấu trúc (ví dụ `crime`, `article_id`, `clause`, `penalty`) và thêm chỉ dẫn “không suy diễn hay thay số Điều; phải trích nguyên `article_id` từ graph fact”. Có thể hậu kiểm câu trả lời: nếu số Điều xuất hiện trong câu trả lời không nằm trong Article facts đã truy xuất thì yêu cầu LLM sửa lại. Hậu kiểm tăng độ trễ/thêm một lần gọi LLM, nhưng giảm hallucination pháp lý.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> GraphRAG phù hợp khi câu hỏi cần nối nhiều nguồn hoặc nhiều bước suy luận: ở Q3, GraphRAG tăng từ recall/judge `0.00/0` lên `1.00/2` nhờ đường đi từ vụ án sang Điều 251; Q4–Q6 cũng có recall cao hơn Flat RAG. Trung bình, GraphRAG đạt recall `0.74` và judge `1.50`, cao hơn Flat RAG (`0.43` và `1.00`). Đổi lại, graph tốn `8.46×` chi phí dựng chỉ mục, `3.92×` chi phí mỗi câu và `4.41×` input token, đồng thời vẫn có thể bỏ sót khoản luật hoặc để LLM nhầm số Điều. Vì vậy Flat RAG là đủ cho câu hỏi một nguồn, trực tiếp như Q1 và Q2 (cả hai đều đạt `1.00/2`); nên dùng KG khi giá trị của liên kết thực thể và truy vấn xuyên KB lớn hơn chi phí tăng thêm.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00076. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Trịnh Vũ Kiên

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> …
