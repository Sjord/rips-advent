# Day 4 - Father Christmas

Can you spot the vulnerability?

```java
import javax.servlet.http.*;

public class Login extends HttpServlet {
  protected void doPost(HttpServletRequest request,
                        HttpServletResponse response) {
    String url = request.getParameter("url");
    //only relative urls are allowed!
    if (url.startsWith("/")) {
      response.sendRedirect(url);
    }
  }
}```

[Show Solution](04-father-christmas-solution.md)
