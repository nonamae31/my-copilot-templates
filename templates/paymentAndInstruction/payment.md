# VNPay Payment Integration — SaveFood

## VNPayService Implementation

```csharp
// Infrastructure/Services/VNPayService.cs
public class VNPayService(IConfiguration config) : IVNPayService
{
    public string CreatePaymentUrl(Order order, HttpContext context)
    {
        var vnpay = new SortedList<string, string>(StringComparer.OrdinalIgnoreCase)
        {
            ["vnp_Version"]    = "2.1.0",
            ["vnp_Command"]    = "pay",
            ["vnp_TmnCode"]    = config["VNPay:TmnCode"]!,
            ["vnp_Amount"]     = ((long)(order.TotalAmount * 100)).ToString(),
            ["vnp_CurrCode"]   = "VND",
            ["vnp_TxnRef"]     = order.Id.ToString("N"),
            ["vnp_OrderInfo"]  = $"SaveFood Order {order.Id}",
            ["vnp_OrderType"]  = "other",
            ["vnp_Locale"]     = "vn",
            ["vnp_ReturnUrl"]  = config["VNPay:ReturnUrl"]!,
            ["vnp_IpAddr"]     = context.Connection.RemoteIpAddress?.ToString() ?? "127.0.0.1",
            ["vnp_CreateDate"] = DateTime.UtcNow.AddHours(7).ToString("yyyyMMddHHmmss"),
            ["vnp_ExpireDate"] = DateTime.UtcNow.AddHours(7).AddMinutes(15).ToString("yyyyMMddHHmmss"),
        };

        var queryString = string.Join("&", vnpay.Select(kv => $"{kv.Key}={Uri.EscapeDataString(kv.Value)}"));
        var secureHash = HmacSha512(config["VNPay:HashSecret"]!, queryString);
        return $"{config["VNPay:BaseUrl"]}?{queryString}&vnp_SecureHash={secureHash}";
    }

    public VNPayResponse ProcessCallback(IQueryCollection query)
    {
        var fields = query
            .Where(kv => kv.Key.StartsWith("vnp_") && kv.Key != "vnp_SecureHash")
            .OrderBy(kv => kv.Key)
            .ToDictionary(kv => kv.Key, kv => kv.Value.ToString());

        var queryString = string.Join("&", fields.Select(kv => $"{kv.Key}={Uri.EscapeDataString(kv.Value)}"));
        var computedHash = HmacSha512(config["VNPay:HashSecret"]!, queryString);
        var receivedHash = query["vnp_SecureHash"].ToString();

        return new VNPayResponse
        {
            IsSuccess = computedHash.Equals(receivedHash, StringComparison.OrdinalIgnoreCase)
                        && query["vnp_ResponseCode"] == "00",
            OrderId = Guid.TryParse(query["vnp_TxnRef"], out var id) ? id : Guid.Empty,
            TransactionId = query["vnp_TransactionNo"].ToString(),
            Amount = long.TryParse(query["vnp_Amount"], out var amt) ? amt / 100m : 0,
            ResponseCode = query["vnp_ResponseCode"].ToString()
        };
    }

    private static string HmacSha512(string key, string data)
    {
        using var hmac = new HMACSHA512(Encoding.UTF8.GetBytes(key));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(data));
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }
}
```

## PaymentController

```csharp
[ApiController]
[Route("api/payment")]
public class PaymentController(IOrderService orderService, IVNPayService vnpayService) : ControllerBase
{
    [HttpPost("create-vnpay/{orderId:guid}")]
    [Authorize(Roles = "Customer")]
    public async Task<IActionResult> CreateVNPayUrl(Guid orderId)
    {
        var order = await orderService.GetOrderEntityAsync(orderId);
        var url = vnpayService.CreatePaymentUrl(order, HttpContext);
        return Ok(new { paymentUrl = url });
    }

    [HttpGet("vnpay-return")]
    public async Task<IActionResult> VNPayReturn()
    {
        var response = vnpayService.ProcessCallback(Request.Query);
        if (response.IsSuccess)
            await orderService.ConfirmPaymentAsync(response.OrderId, response.TransactionId);
        else
            await orderService.CancelOrderAsync(response.OrderId, "Payment failed");

        // Redirect to frontend with result
        var frontendUrl = response.IsSuccess
            ? $"https://savefood.app/payment/success?orderId={response.OrderId}"
            : $"https://savefood.app/payment/failed?orderId={response.OrderId}";
        return Redirect(frontendUrl);
    }
}
```

## VNPayResponse DTO

```csharp
public record VNPayResponse
{
    public bool IsSuccess { get; init; }
    public Guid OrderId { get; init; }
    public string TransactionId { get; init; } = string.Empty;
    public decimal Amount { get; init; }
    public string ResponseCode { get; init; } = string.Empty;
}
```
