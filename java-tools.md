# java工具

## 验证lib目录的jar文件是否有错误

maven下载有的时候jar文件的完整性不能得到保证，部署安装的时候，可能报错，java.util.zip.ZipException: invalid LOC header (bad signature)

你可以使用下面的java程序来验证那个jar文件有错误。

把这个java文件放到lib目录下，并使用 javac JarFileValidator.java编译， [ java JarFileValidator . ] 运行。遇到第1个验证失败的jar停止。

```java
/**
 * jar文件验证器
 * @author Administrator
 *
 */
public class JarFileValidator {

	/**
	 * 得到文件的扩展名
	 * @param path
	 * @return
	 */
	public static String getFilenameExtension(String path) {
		if (path == null) {
			return null;
		}
		int extIndex = path.lastIndexOf(".");
		if (extIndex == -1) {
			return null;
		}
		int folderIndex = path.lastIndexOf(".");
		if (folderIndex > extIndex) {
			return null;
		}
		return path.substring(extIndex + 1);
	}

	public static byte[] getAllBytes(InputStream is) throws IOException {
		if (is != null) {
			ByteArrayOutputStream out = new ByteArrayOutputStream();
			int bufSize = 1024;
			byte[] buf = new byte[bufSize];
			int res = 0;
			while ((res = is.read(buf)) > 0) {
				out.write(buf, 0, res);
			}
			return out.toByteArray();
		}
		return null;
	}

	public static void main(String[] args) throws IOException {
		File dir = new File(args[0]);
		for (File file : dir.listFiles()) {
			if (getFilenameExtension(file.getPath()).equals("jar")) {
				ZipFile zipFile = new ZipFile(file, Charset.forName("UTF-8"));
				System.out.print(file + " start validate...");
				Enumeration<? extends ZipEntry> entries = zipFile.entries();
				while (entries.hasMoreElements()) {
					ZipEntry en = entries.nextElement();
					if (!en.isDirectory()) {
						InputStream is = zipFile.getInputStream(en);
						byte[] bytes = getAllBytes(is);
						bytes = null;
						is.close();
					}
				}
				zipFile.close();
				System.out.println(", success.");
			}
		}
	}

}
```

