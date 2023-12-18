KMP算法是一种**字符串匹配**算法，可以在 O(n+m) 的时间复杂度内实现两个字符串的匹配。

##### 字符串匹配问题

> 字符串 P 是否为字符串 S 的字串？如果是，它出现在 S 的哪些位置？

其中 S 称为主串，P 称为模式串

主串：abbaabbaaba

模式串：abbaaba

模式串与主串从头开始匹配，发现第7个字符不匹配

通常，主串会回退到第二个字符，模式串回退到开头，然后重新开始比较



可以做一些优化，**发生不匹配时，主串不回退**

主串若想要与模式串匹配，必须要匹配7个长度的字符；因此即使主串回退到第2个字符，在后续匹配流程中，肯定会重新匹配到主串的第7个字符。

**当在主串的某个字符 c 上发生不匹配时，主串即使回退，最终还是会重新匹配到字符 c 上**

因此，干脆主串不进行回退

**主串不回退，模式串必须尽可能少的回退，并且模式串回退位置的前面那段已经和主串匹配，这样主串才能不回退**



查找模式串回退位置？

不匹配发生时，前面匹配的一小段 abbaab 是相同的，这样的话用 abbaab 的头部去匹配 abbaab 的尾部，那最长的那段就是答案

> ```text
> abbaab 的头部有 a, ab, abb, abba, abbaa（不包含最后一个字符。下文称之为「真前缀」）
> abbaab 的尾部有 b, ab, aab, baab, bbaab（不包含第一个字符。下文称之为「真后缀」）
> 这样最长匹配是 ab。也就是说主串不回退时，模式串需要回退到第三个字符去和主串继续匹配。
> ```

要计算的内容只和模式串有关，那就假设模式串在其所有位置上都发生了不匹配，模式串在和主串匹配前把其所有位置的最长匹配都算出来（算个长度就行），生成一张表，之后发生不匹配时直接查这张表就行。

**所有要与主串匹配的字符串，必须先自身匹配：对每个子字符串 [0...i]，算出其「相匹配的真前缀与真后缀中，最长的字符串的长度」。**

abababzabababa 最大匹配数为子字符串 [0...i] 的最长匹配的长度

```
子字符串　 a b a b a b z a b a b a b a
最大匹配数 0 0 1 2 3 4 0 1 2 3 4 5 6 ?
```

对子字符串 abababzababab 来说，

真前缀有 a, ab, aba, abab, ababa, [ababab](https://www.zhihu.com/search?q=ababab&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A37475572}), abababz, ...

真后缀有 b, ab, bab, abab, babab, ababab, [zababab](https://www.zhihu.com/search?q=zababab&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A37475572}), ...

所以子字符串 abababzababab 的真前缀和真后缀最大匹配了 6 个（ababab），那**次大**匹配了多少呢？

容易看出次大匹配了 4 个（abab），更仔细地观察可以发现，**次大匹配必定在最大匹配 ababab 中，所以次大匹配数就是 ababab 的最大匹配数！**

直接去查算出的表，可以得出该值为 4。

第三大的匹配数同理，它既然比 4 要小，那真前缀和真后缀也只能在 abab 中找，即 abab 的最大匹配数，查表可得该值为 2。

再往下就没有更短的匹配了。

回顾完毕，来计算 ? 的值：既然末尾字母不是 z，那么就不能直接 6+1=7 了，我们回退到次大匹配 abab，刚好 abab 之后的 a 与末尾的 a 匹配，所以 ? 处的最大匹配数为 5。

计算最大匹配数代码

```
// 构造模式串 pattern 的最大匹配数表
int[] calculateMaxMatchLengths(String pattern) {
	int[] maxMatchLengths = new int[pattern.length()];
	int maxLength = 0;
	for (int i = 1; i < pattern.length(); i++) {
		while (maxLength > 0 && pattern.charAt(maxLength) != pattern.charAt(i)) {
			maxLength = maxMatchLengths[maxLength - 1];
		}
		if (pattern.charAt(i) == pattern.charAt(maxLength)) {
			maxLength++;
		}
		maxMatchLengths[i] = maxLength;
	}
	return maxMatchLengths;
}
```

KMP 匹配过程

```
// 在文本 text 中寻找模式串 pattern，返回所有匹配的位置开头
List<Integer> search(String text, String pattern) {
	List<Integer> positions = new ArrayList<>();
	int[] maxMatchLengths = calculateMaxMatchLengths(pattern);
	int count = 0;
	for (int i = 0; i < text.length(); i++) {
		while (count > 0 && pattern.charAt(count) != text.charAt(i)) {
            count = maxMatchLengths[count - 1];
        }
        if (pattern.charAt(count) == text.charAt(i)) {
            count++;
        }
        if (count == pattern.length()) {
            positions.add(i - pattern.length() + 1);
            count = maxMatchLengths[count - 1];
        }
	}
	
	return positions;
}
```

