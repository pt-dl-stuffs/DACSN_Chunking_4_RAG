# Phân tích các phương pháp Chunking cho hệ thống RAG

## 1. Mục tiêu

Đề tài tìm hiểu cách chia tài liệu thành các đoạn nhỏ (Chunk) trước khi đưa vào hệ thống Retrieval-Augmented Generation (RAG). Mục tiêu là cài đặt, phân tích độ phức tạp và so sánh ảnh hưởng của các phương pháp Chunking đến khả năng truy xuất thông tin.

## 2. Dữ liệu và cách đo

- Sử dụng 3 tài liệu tiếng Việt trong thư mục `sources/` và bộ 49 câu hỏi
- Mỗi câu hỏi lưu vị trí đoạn đáp án trong văn bản nguồn để làm bằng chứng tham chiếu
- Tokenizer sẽ tính mỗi cụm ký tự liên tiếp không chứa khoảng trắng là một token. Đây là quy ước thống nhất cho thí nghiệm, không phải tokenizer BPE/WordPiece của một mô hình cụ thể
- Mỗi Chunk lưu tài liệu nguồn, nội dung, số token và vị trí trong văn bản gốc (không bao gồm ký tự tại vị trí kết thúc)

## 3. Các phương pháp được cài đặt

1. **Fixed-size:** chia văn bản theo giới hạn token cố định và có Overlap để giữ lại một phần ngữ cảnh giữa hai Chunk
2. **Sentence-based:** ghép các câu liên tiếp đến khi gần đạt giới hạn; câu quá dài được chia nhỏ để không vượt giới hạn
3. **Recursive:** ưu tiên chia theo đoạn văn, xuống dòng, dấu câu và khoảng trắng; giữ dấu phân cách, sau đó chia theo token nếu vẫn cần thiết
4. **Semantic:** tạo Embedding cho từng câu, so sánh Cosine Similarity giữa các câu liền kề và chỉ ghép khi đạt ngưỡng, đồng thời không vượt giới hạn kích thước Chunk

Cấu hình mặc định: tối đa **180 token/Chunk**, Overlap **30 token** cho Fixed-size và ngưỡng Semantic **0.65**. Các giá trị này là cấu hình ban đầu để thực nghiệm, không được xem là tối ưu cho mọi tài liệu.

## 4. Đánh giá chất lượng truy xuất

Các phương pháp được đánh giá trên cùng bộ câu hỏi và cùng retriever trong mỗi lần chạy. 
Chunk tham chiếu được xác định từ vị trí đáp án trong văn bản gốc, không so sánh Chunk ID giữa các phương pháp. Với mỗi câu hỏi và mỗi phương pháp, chương trình chọn tối đa một Chunk có phần giao với đáp án lớn nhất; Chunk phải bao phủ ít nhất **50%** đoạn đáp án mới được xem là liên quan. Cách này hạn chế việc Overlap làm một bằng chứng bị tính thành nhiều Chunk đúng.

Các chỉ số được tính gồm **Precision@K**, **Recall@K**, **MRR** và **Answer-span Coverage@K**. Coverage đo tỷ lệ ký tự đáp án được các Chunk trong Top-K bao phủ và không đếm lặp vùng Overlap.

## 5. Phân tích độ phức tạp

Gọi `C` là số ký tự
    `T` là số token
    `S` là số câu
    `d` là số chiều Embedding
    `q` là số cấp dấu phân cách
    `T_emb` là thời gian tạo Embedding

- **Fixed-size:** xử lý tuyến tính theo văn bản; với kích thước và Overlap cố định, thời gian điển hình là `O(C)`.
- **Sentence-based:** tách câu, đếm token và ghép câu qua văn bản; thời gian điển hình là `O(C)`.
- **Recursive:** mỗi cấp phân cách quét/đếm các phần văn bản; mô hình hóa là `O(qC)`, thường rút gọn thành `O(C)` khi `q` cố định.
- **Semantic:** gồm xử lý văn bản, tạo Embedding và so sánh vector; có thể biểu diễn là `O(C + T_emb + Sd)`.

Các ước lượng trên được đối chiếu với thời gian chạy thực tế. Chi phí đếm token, tạo Chunk, lưu kết quả và tạo Embedding không bị bỏ qua khi phân tích
