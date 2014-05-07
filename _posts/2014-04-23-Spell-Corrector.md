---
layout: post
title: 拼写错误修正(java)
categories: [nlp]
tags: [nlp, bayes]
description: 对输入词进行转换，将编辑距离在3以内的词作为候选词，然后根据这些词的tfidf进行排序，将最高的词输出，作为修正后的词
---
###下面是拼写检查器很好的文章，本文参照[该文](http://blog.youxu.info/spell-correct.html)，将实现java版
整个拼写检查器的基础就是贝叶斯概率模型
我简单的介绍一下它的工作原理. 给定一个单词, 我们的任务是选择和它最相似的拼写正确的单词. (如果这个单词本身拼写就是正确的, 那么最相近的就是它自己啦). 当然, 不可能绝对的找到相近的单词, 比如说给定 lates 这个单词, 它应该别更正为 ``late`` 呢 还是 ``latest`` 呢? 这些困难指示我们, 需要使用概率论, 而不是基于规则的判断. 我们说, 给定一个词 `w`, 在所有正确的拼写词中, 我们想要找一个正确的词 `c`, 使得对于 `w` 的条件概率最大, 也就是说:
*<center>argmaxc P(c|w)</center>*
----
{{ site.excerpt_separator }}
按照贝叶斯理论上面的式子等价于:
*<center>argmaxc P(w|c)P(c) / P(w)</center>*
----
因为用户可以输错任何词, 因此对于任何 `c` 来讲, 出现 `w` 的概率 `P(w)` 都是一样的, 从而我们在上式中忽略它, 写成:
*<center>argmaxc P(w|c)P(c)</center>*
----
####这个式子有三个部分, 从右到左, 分别是:
1. `P(c)`, 文章中出现一个正确拼写词 `c` 的概率, 也就是说, 在英语文章中, `c` 出现的概率有多大呢? 因为这个概率完全由英语这种语言决定, 我们称之为做语言模型. 好比说, 英语中出现 ``the`` 的概率  `P('the')` 就相对高, 而出现  `P('zxzxzxzyy')` 的概率接近0(假设后者也是一个词的话).
2. `P(w|c)`, 在用户想键入 `c` 的情况下敲成 `w` 的概率. 因为这个是代表用户会以多大的概率把 `c` 敲错成 `w`, 因此这个被称为误差模型.
3. argmaxc, 用来枚举所有可能的 `c` 并且选取概率最大的, 因为我们有理由相信, 一个(正确的)单词出现的频率高, 用户又容易把它敲成另一个错误的单词, 那么, 那个敲错的单词应该被更正为这个正确的.

有人肯定要问, 你笨啊, 为什么把最简单的一个 `P(c|w)` 变成两项复杂的式子来计算? 答案是本质上 `P(c|w)` 就是和这两项同时相关的, 因此拆成两项反而容易处理. 举个例子, 比如一个单词 ``thew`` 拼错了. 看上去 ``thaw`` 应该是正确的, 因为就是把 `a` 打成 `e` 了. 然而, 也有可能用户想要的是 ``the``, 因为 ``the`` 是英语中常见的一个词, 并且很有可能打字时候手不小心从 `e` 滑到 `w` 了. 因此, 在这种情况下, 我们想要计算  `P(c|w)`, 就必须同时考虑 `c` 出现的概率和从 `c` 到 `w` 的概率. 把一项拆成两项反而让这个问题更加容易更加清晰.

###使用到的语料库为:big.txt.
[代码 github](github：https://github.com/dreamboy127/Spell_java)
----

####java 代码如下:
```java
import java.util.*;  
import java.io.*;  
public class SpellCorrect{  
    public static void readLines(String file, ArrayList<String> lines) {  
        BufferedReader reader = null;  
        try {  
            reader = new BufferedReader(new FileReader(new File(file)));  
            String line = null;  
            while ((line = reader.readLine()) != null) {  
                lines.add(line);                                                              
            }                                                      
        } catch (FileNotFoundException e) {  
            e.printStackTrace();                 
        } catch (IOException e) {  
            e.printStackTrace();  
        } finally {  
            if (reader != null) {  
                try {  
                    reader.close();  
                } catch (IOException e) {  
                    e.printStackTrace();                                                            
                }  
            }  
        }                      
    }  
        private static String readText(File file) {   
                String text = null;  
                try  
                {  
                        InputStreamReader read = new InputStreamReader(new FileInputStream(file));  
                        BufferedReader br = new BufferedReader(read);      
                        StringBuffer buff = new StringBuffer();       
                        while((text = br.readLine()) != null)  
                {      
                                buff.append(text + "\r\n");      
                }      
                br.close();           
                text = buff.toString();   
            }    
                catch(FileNotFoundException e)    
            {     
                        System.out.println(e);    
            }    
            catch(IOException e)    
            {     
                System.out.println(e);    
            }  
               return text;  
        }  
    public static void tokenizeAndLowerCase(String line, ArrayList<String> tokens) {  
    // TODO Auto-generated method stub  
        StringTokenizer strTok = new StringTokenizer(line,"\r\n\t/\\\':\" ()[]{};.,#-_=!@$%^&*+1234567890");  
        while (strTok.hasMoreTokens()) {  
            String token = strTok.nextToken();  
            tokens.add(token.toLowerCase().trim());  
        }  
    }  
    public static void trainPrior(ArrayList<String> str,Map<String,Integer> map)  
    {  
        for(int i=0;i<str.size();i++)  
        {  
            if(map.containsKey(str.get(i)))  
            {  
                int tmp=map.get(str.get(i));  
                map.put(str.get(i),1+tmp);  
            }  
            else  
                map.put(str.get(i),1);  
        }  
    }  
    public static Set<String> Edit1(String str){  
        Set<String> array=new HashSet<String>();  
        for(int i=0;i<str.length();i++)//delete  
        {  
            String tmpstr=str.substring(0,i)+str.substring(i+1,str.length());  
            array.add(tmpstr);  
        }  
          
        for(int i=0;i<str.length();i++)//insert  
        {  
            for(char x='a';x<='z';x++)  
            {  
                String tmpstr=str.substring(0,i)+x+str.substring(i,str.length());  
                array.add(tmpstr);  
            }     
        }  
        for(int i=0;i<str.length()-1;i++)//trans  
        {  
            String tmpstr=str.substring(0,i)+str.charAt(i+1)+str.charAt(i)+str.substring(i+2,str.length());  
            array.add(tmpstr);  
        }  
        for(int i=0;i<str.length();i++)//convert  
        {  
            for(char x='a';x<='z';x++)  
            {  
                String tmpstr=str.substring(0,i)+x+str.substring(i+1,str.length());  
                array.add(tmpstr);  
            }  
        }  
        return array;  
    }  
    public static Set<String> Edit2(String str){  
        Set<String> array=new HashSet<String>();  
        array=Edit1(str);  
        Set<String> array2=new HashSet<String>();  
        Iterator<String> iter=array.iterator();  
        while(iter.hasNext())  
        {  
            String str1=iter.next();  
            array2.addAll(Edit1(str1));  
        }  
        return array2;  
    }  
    public static boolean kowns(Set<String> checkset,Set<String> wordset)  
    {  
        Iterator<String> iter=checkset.iterator();  
        while(iter.hasNext())  
        {  
            String str=iter.next();  
            if(!wordset.contains(str))  
                iter.remove();  
        }  
        return checkset.size()>0;  
    }  
    public static void main(String[] args){  
        String text=readText(new File("big.txt"));  
        ArrayList<String> s=new ArrayList<String>();  
        tokenizeAndLowerCase(text,s);  
        Map<String,Integer> map=new HashMap<String,Integer>();  
        trainPrior(s,map);  
        Set<String> keys=map.keySet();  
 //       System.out.println(map.size());  
        Scanner scan=new Scanner(System.in);  
        System.out.println("spell correct starting");  
        while(true)  
        {  
            System.out.println("please input a term:");  
            String str=scan.next();  
            if("q".equals(str))  
                break;  
            Set<String> edit1=Edit1(str);  
            Set<String> edit2=Edit2(str);  
            boolean flag=kowns(new HashSet<String>(Arrays.asList(str)),keys);  
            if(flag)  
                return;  
            Set<String> edit=edit1;  
            flag=kowns(edit,keys);  
            if(!flag)  
            {  
                edit=edit2;  
                flag=kowns(edit,keys);  
            }  
            Iterator<String> iter=edit.iterator();  
            int max=0;  
            int tmp=1;  
            String maxStr=null;  
            while(iter.hasNext())  
            {  
                String tmpStr = iter.next();  
//            System.out.println(tmpStr);  
                tmp=map.get(tmpStr);  
  
                if(max<tmp)  
                {  
                    maxStr=tmpStr;  
                    max=tmp;  
                }  
            }  
            System.out.println(maxStr);  
        }  
        System.out.println("spell correct ending");  
//        Set<Map.Entry<String,Integer>> allSet=null;  
//        allSet=map.entrySet();  
//        for(Map.Entry<String,Integer> me : allSet)  
//            System.out.println(me.getKey()+"-->"+me.getValue());  
    }  
}
```
