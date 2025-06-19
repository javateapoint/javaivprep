


|             Status            | Meaning & Typical Use                                                        | Example                                                                                 |
| :---------------------------: | ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
|           **200 OK**          | Request succeeded; response body contains the representation requested.      | `GET /users/123` → returns user JSON                                                    |
|        **201 Created**        | Resource successfully created; `Location` header points to the new resource. | `POST /orders` → `201 Created` + `Location: /orders/789`                                |
|        **202 Accepted**       | Request accepted for processing but not yet completed (async).               | `POST /reports` → `202 Accepted` + `Location: /jobs/456/status`                         |
|       **204 No Content**      | Request succeeded but there’s no content to return (e.g. delete).            | `DELETE /sessions/abc` → `204 No Content`                                               |
|      **400 Bad Request**      | Client request malformed or validation failed.                               | `POST /users` with invalid email → `400 Bad Request` + error details in response body   |
|      **401 Unauthorized**     | Authentication required or failed (invalid/expired token).                   | `GET /protected` without token → `401 Unauthorized` + `WWW-Authenticate` header         |
|       **403 Forbidden**       | Authenticated but not permitted to perform this action.                      | `DELETE /users/123` by a non-admin → `403 Forbidden`                                    |
|       **404 Not Found**       | Resource does not exist.                                                     | `GET /products/9999` → `404 Not Found`                                                  |
|        **409 Conflict**       | Request could not be processed due to conflict (e.g. duplicate).             | `POST /users` with existing username → `409 Conflict` + explanation                     |
|  **422 Unprocessable Entity** | Semantic validation error on well‑formed request.                            | `POST /orders` with out‑of‑stock item → `422 Unprocessable Entity` + validation details |
| **500 Internal Server Error** | Unexpected server error or unhandled exception.                              | Any unhandled exception → `500 Internal Server Error`                                   |
|  **503 Service Unavailable**  | Service temporarily down for maintenance or overloaded.                      | Heavy load or maintenance window → `503 Service Unavailable` + retry‑after header       |
|    **504 Gateway Timeout**    | Upstream service did not respond in time.                                    | API gateway timeout calling legacy service → `504 Gateway Timeout`                      |



## When to Use 202 in Event‑Driven Architectures

```java
@RestController
public class OrderController {
  private final ApplicationEventPublisher publisher;

  public OrderController(ApplicationEventPublisher publisher) {
    this.publisher = publisher;
  }

  @PostMapping("/orders")
  public ResponseEntity<Void> createOrder(@RequestBody OrderDto dto) {
    // 1. Validate/persist the incoming DTO
    Order order = orderService.save(dto);
    // 2. Publish an internal Spring event or to a message broker
    publisher.publishEvent(new OrderCreatedEvent(order.getId()));
    // 3. Immediately return 202 → work continues asynchronously
    return ResponseEntity
             .accepted()
             .header(HttpHeaders.LOCATION, "/orders/" + order.getId() + "/status")
             .build();
  }
}


```
