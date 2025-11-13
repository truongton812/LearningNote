Giải thích thêm về handler: Handler là điểm vào (entry point) của Lambda function, được AWS Lambda gọi khi function được kích hoạt bởi một sự kiện (event).

Tại sao nên khởi tạo kết nối database bên ngoài handler?

Lambda Execution Context Reuse

- AWS Lambda giữ lại execution environment (bộ nhớ, biến global) sau khi function chạy xong để tái sử dụng cho các lần invoke tiếp theo (warm start).
- Code bên ngoài handler (ví dụ: khởi tạo DB connection) chỉ chạy một lần khi execution environment được khởi tạo.
- Code bên trong handler chạy mỗi lần invoke, dẫn đến việc kết nối DB được tạo mới liên tục → tăng latency và tốn tài nguyên.


https://docs.google.com/document/d/103CadDRznUzqfriz7jC2cOY4Lfx6EzEQi-Xyw_n8tJw/edit?tab=t.0



---


`
import json

def lambda_handler(event, context) -> None:
    # TODO implement
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
`

Dòng code này định nghĩa hàm handler chính lambda_handler cho một AWS Lambda function bằng Python.

Cụ thể:

`def lambda_handler(event, context) -> None:` Đây là điểm vào (entry point) mặc định khi AWS Lambda thực thi hàm. Lambda runtime sẽ gọi hàm này, truyền vào hai đối số:

- event: dữ liệu sự kiện (event data) truyền đến Lambda khi nó được kích hoạt, thường là một dict Python chứa các thông tin về sự kiện.

- context: đối tượng cung cấp thông tin về ngữ cảnh thực thi Lambda như thời gian chạy còn lại, request ID, v.v.

Dấu -> None là kiểu trả về (type hint) cho biết hàm không trả về giá trị nào.

Dòng client = boto3.client('ce', region_name='us-east-1') khởi tạo một client AWS SDK (boto3) để gọi dịch vụ Cost Explorer (ce) tại region us-east-1. Client này dùng để truy vấn dữ liệu chi phí AWS trong hàm Lambda.

Dấu -> None không phải là comment mà là một cú pháp gọi là "type hint" trong Python, dùng để chỉ kiểu dữ liệu trả về của hàm. Cụ thể:

-> None nghĩa là hàm này không trả về giá trị nào (tương đương trả về None).

Đây chỉ là thông tin cho người đọc và các công cụ kiểm tra kiểu tĩnh (type checkers, IDE), không ảnh hưởng đến việc chạy của chương trình.

Nếu có hoặc không có -> None thì hàm vẫn chạy bình thường, nên về mặt chức năng nó không bắt buộc nhưng giúp code rõ ràng và dễ bảo trì hơn.

Tóm lại, đây không phải comment vô nghĩa mà là lời chú thích kiểu trả về hàm, hữu ích cho việc đọc hiểu và kiểm

---

Trong AWS Lambda, hàm xử lý (handler) của bạn có thể trả về một giá trị (return) để gửi phản hồi lại cho dịch vụ hoặc client gọi Lambda đó. Việc cần return trong hàm Lambda có các lý do chính sau:

- Khi Lambda được gọi theo kiểu đồng bộ (synchronous invocation), giá trị trả về từ hàm Lambda sẽ được chuyển lại cho client hoặc dịch vụ gọi Lambda trong định dạng JSON. Nếu không có return, giá trị trả về sẽ là null.

- Việc trả về dữ liệu giúp client biết kết quả của lần gọi Lambda, ví dụ như trạng thái thực thi, dữ liệu đã xử lý, hoặc thông báo lỗi.

- Nếu Lambda được dùng làm backend cho API Gateway hay các dịch vụ khác, giá trị trả về thường là cấu trúc JSON chứa mã trạng thái HTTP, header, và body để giao tiếp với client.

- Trả về giá trị cũng giúp việc test, debug dễ dàng hơn vì ta có thể kiểm tra phản hồi được trả.

Ngược lại, nếu Lambda được gọi bất đồng bộ (asynchronous invocation), giá trị trả về thường bị bỏ qua, nên return có thể không cần thiết, nhưng trong nhiều trường hợp bạn vẫn nên trả về kết quả chuẩn để mở rộng tính năng về sau hoặc khi chuyển sang gọi đồng bộ.

Tóm lại, return trong Lambda là để cung cấp phản hồi cụ thể cho dịch vụ hoặc client gọi hàm, cải thiện khả năng tương tác và quản lý kết quả thực thi hàm
