# Day 11 - Carolers

Can you spot the vulnerability?

```java
import javax.servlet.http.*;
import java.io.*;
import java.nio.file.Files;
import org.apache.commons.compress.archivers.ArchiveStreamFactory;
import org.apache.commons.compress.archivers.tar.*;
import org.apache.commons.io.IOUtils;

public class ExtractFiles extends HttpServlet {
  private static void extract() throws Exception {
    // /tmp/uploaded.tar is user controlled and an uploaded file.
    final InputStream is = new FileInputStream(new File("/tmp/uploaded.tar"));
    final TarArchiveInputStream tarInputStream = (TarArchiveInputStream) (new ArchiveStreamFactory().createArchiveInputStream(ArchiveStreamFactory.TAR, is));
    File tmpDir = Files.createTempDirectory("test").toFile();
    TarArchiveEntry entry;
    while ((entry = tarInputStream.getNextTarEntry()) != null) {
      File file = new File(tmpDir, entry.getName().replace("../", ""));
      if (entry.isDirectory()) {
        file.mkdirs();
      } else {
        IOUtils.copy(tarInputStream, new FileOutputStream(file));
      }
    }
    is.close();
    tarInputStream.close();
  }
}
```

[Show Solution](11-carolers-solution.md)
