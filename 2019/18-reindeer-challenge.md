# Day 18 - Reindeer

Can you spot the vulnerability?

```java
import org.apache.tomcat.util.http.fileupload.IOUtils;
import javax.servlet.http.*;
import java.util.HashMap;

public class LoadConfig extends HttpServlet {
  public static HashMap<String, String> parseRequest(String value) {
    HashMap<String, String> result = new HashMap<String, String>();
    if (value != null) {
      String tmp[] = value.split("@");
      for (int i = 0; i < tmp.length; i = i + 2) {
        result.put(tmp[i], tmp[i + 1]);
      }
    }
    return result;
  }

  protected void doGet(HttpServletRequest request, HttpServletResponse response) {
    if (request.getParameter("home") != null) {
      HttpSession session = request.getSession(true);
      if (!session.isNew()){
        if (validBasicAuthHeader()) { // Checks the Basic Authorization header (password check)
          // Execute last command:
          ProcessBuilder processBuilder = new ProcessBuilder();
          processBuilder.command((String)session.getAttribute("last_command"));
          try {
            Process process = processBuilder.start();
            IOUtils.copy(process.getInputStream(), response.getOutputStream());
          }
          catch (Exception e){
            return;
          }
        }
      }
    } else if (request.getParameter("save_session") != null) {
      String value = request.getParameter("config");
      HashMap<String, String> config = parseRequest(value);
      for (String i : config.keySet()) {
        Cookie settings = new Cookie(i, config.get(i));
        response.addCookie(settings);
      }
    } else {
      HttpSession session = request.getSession(true);
      if (session.isNew()) {
        HashMap<String, String> whitelist = new HashMap<String, String>();
        whitelist.put("home", "yes");
        whitelist.put("role", "frontend");

        String value = request.getParameter("config");
        HashMap<String, String> config = parseRequest(value);

        whitelist.putAll(config);
        for (String i : whitelist.keySet()) {
          session.setAttribute(i, whitelist.get(i));
        }
      }
    }
  }
}
```

[Show Solution](18-reindeer-solution.md)
