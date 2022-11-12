# Day 10 - Anticipation

Can you spot the vulnerability?

```java
@RequestMapping("/webdav")
  public void webdav(HttpServletResponse res, @RequestParam("name") String name) throws IOException {
    res.setContentType("text/xml");
    res.setCharacterEncoding("UTF-8");
    PrintWriter pw = res.getWriter();
    name = name.replace("]]", "");
    pw.print("<person>");
    pw.print("<name><![CDATA[" + name.replace(" ","") + "]]></name>");
    pw.print("</person>");
  }
```

[Show Solution](10-anticipation-solution.md)
