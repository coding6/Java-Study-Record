# Java Design Mode
## Strategy Mode

```java
@Data
@AllArgsConstructor
public enum PayType {
    WECHAT_PAY(10, "微信支付"),
    AILPAY_PAY(20, "支付宝支付"),
    UNION_PAY(30, "云闪付支付");

    private int payTypeId;

    private String description;

    public static PayType getTypeFromTypeId(int typeId) {
        for(PayType type : PayType.values()) {
            if (type.getPayTypeId() == typeId) {
                return type;
            }
        }
        return null;
    }
}
```
```java
@Data
@AllArgsConstructor
public class UserPayRequest{
    private Long payAmount;
    private Long userId;
    private Integer typeId;
}
```
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface PayTypeAn{
    PayType value();
}
```
```java
public interface PayRelated{
    /**
     * 通用支付请求
     */
    boolean pay(UserPayRequest request);
}
```
```java
@Component
@PayTypeAn(value = PayType.WECHAT_PAY)
public class WeChatPay implements PayRelated{

    @Override
    boolean pay(UserPayRequest request) {
        /*invoking WeChat-pay API*/
        return true;
    }
}
```
```java
@Component
@PayTypeAn(value = PayType.AILPAY_PAY)
public class AilPay implements PayRelated{

    @Override
    boolean pay(UserPayRequest request) {
        /*invoking AilPay-pay API*/
        return true;
    }
}
```
```java
@Component
public class PayRelatedFactory{
    @Autowired
    private ApplicationContext context;

    private static Map<PayType, PayRelated> beanCache = new HashMap();

    @PostConstruct
    public void init() {
        List<PayRelated> list = new ArrayList(context.getBeansOfType(PayRelated.class).values());
        for (PayRelated payRelated : list) {
            PayType type = payRelated.getClass()..getAnnotation(PayTypeAn.class).value()
            beanCache.put(type, payRelated);
        }
        
    }

    public static PayRelated getPayRelatedByPayType(PayType payType) {
        return beanCache.get(payType);
    }
}
```
```java
 public class UserPayService{

     public boolean pay(UserPayRequest request) {
         PayType type = PayType.getTypeFromTypeId(request.getTypeId());
         PayRelated payRelated = PayRelatedFactory.getPayRelatedByPayType(type);
         if (Objects.nonNull(payRelated)) {
            return payRelated.pay(request);
         }
         return false
     }
 }
```
