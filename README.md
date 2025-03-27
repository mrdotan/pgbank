```mermaid
sequenceDiagram
    participant Consumer (app)
    participant Internal API Gateway
    participant ESB
    participant Backend

    Consumer (app)->>Internal API Gateway: SELECT /accounts/{account-id}<br>Header: X-Request-Id, Authorization: Basic (base64)
    Internal API Gateway->>Internal API Gateway: Step 1: Kiểm tra bảo mật<br>Input: X-Request-Id, Authorization<br>Output: Hợp lệ/401 Unauthorized
    alt Authorization không hợp lệ
        Internal API Gateway-->>Consumer (app): 401 Unauthorized
        Consumer (app)-->>Consumer (app): Lỗi
    else Authorization hợp lệ
        Internal API Gateway->>Internal API Gateway: Step 2: Operation-switch<br>Input: HTTP Method, Endpoint<br>Output: Hợp lệ/400 Bad Request
        alt Endpoint không hợp lệ
            Internal API Gateway-->>Consumer (app): 400 Bad Request
            Consumer (app)-->>Consumer (app): Lỗi
        else Endpoint hợp lệ
            Internal API Gateway->>Internal API Gateway: Step 3: JSON Schema Validator<br>Input: Body (JSON)<br>Output: Hợp lệ/400 Bad Request
            alt JSON không hợp lệ
                Internal API Gateway-->>Consumer (app): 400 Bad Request
                Consumer (app)-->>Consumer (app): Lỗi
            else JSON hợp lệ
                Internal API Gateway->>ESB: Invoke
                ESB->>ESB: Step 1: Mapping dữ liệu đầu<br>Input: JSON<br>Output: XML/Column<br>Note right of ESB: Tối ưu hoá mapping, caching (Redis/Memcached)
                ESB->>Backend: Dữ liệu đã map
                Backend->>Backend: Xử lý yêu cầu<br>Note right of Backend: Tối ưu hoá truy vấn, caching dữ liệu
                alt Xử lý thành công
                    Backend-->>ESB: Trả về dữ liệu [Table/XML]
                    ESB->>ESB: Step 2: Mapping dữ liệu phản hồi<br>Input: Dữ liệu Backend (XML)<br>Output: JSON response
                    ESB-->>Internal API Gateway: JSON response
                    Internal API Gateway-->>Consumer (app): Trả về kết quả JSON<br>Note left of Internal API Gateway: Caching, Rate limiting, Circuit breaker, Logging
                else Xử lý thất bại
                    Backend-->>ESB: 500 Internal Server Error
                    ESB-->>Internal API Gateway: 500 Internal Server Error
                    Internal API Gateway-->>Consumer (app): 500 Internal Server Error
                    Consumer (app)-->>Consumer (app): Lỗi
                end
            end
        end
    end

