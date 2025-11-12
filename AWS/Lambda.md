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
