### 生成字母、数字、特殊字符验证码

```java
import org.apache.commons.lang3.RandomStringUtils;

import java.util.Random;

/**
 * @author lius
 * @date 2021/3/16
 */
public class VerifyCode {

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
//            String result = makeRandomCode(12);
//            if (result.matches("^(?=.*[0-9])(?=.*[A-Z])(?=.*[a-z])(?=.*[!@#$*?])[0-9a-zA-Z!@#$*?]{12,12}$")) {
//                System.out.println("随机验证码：" + result);
//            } else {
//                System.out.println("无效验证码：" + result);
//            }
//            System.out.println(getRandomCode());

//            System.out.println(RandomStringUtils.random(12, 0, 0, false, false, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890!@#$*?".toCharArray()));;
            System.out.println(getRandomCodeByApacheCommons(12));

        }
    }

    public static String getRandomCode() {
        String result;
        do {
             result = makeRandomCode(12);
            // 包含大写字母、小写字母、数字、特殊字符
            // ^(?=.*[0-9])(?=.*[A-Z])(?=.*[a-z])(?=.*[!@#$*?])[0-9a-zA-Z!@#$*?]{12,12}$
            // 包含大写字母或小写字母、数字、特殊字符
            // ^(?=.*[0-9])((?=.*[A-Z])|(?=.*[a-z]))(?=.*[!@#$*?])[0-9a-zA-Z!@#$*?]{12,12}$
        } while (!result.matches("^(?=.*[0-9])(?=.*[A-Z])(?=.*[a-z])(?=.*[!@#$*?])[0-9a-zA-Z!@#$*?]{12,12}$"));

        return result;
    }

    public static String makeRandomCode(int len) {
        char[] charArray = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890!@#$*?".toCharArray();
        StringBuilder sb = new StringBuilder();
        Random r = new Random();
        for (int i = 0; i < len; ++i) {
            sb.append(charArray[r.nextInt(charArray.length)]);
        }
        return sb.toString();
    }

    // 通过apache commons RandomStringUtils生成
    public static String getRandomCodeByApacheCommons(int len) {
       return RandomStringUtils.random(len, 33, 127, false, false);
    }
}
```

