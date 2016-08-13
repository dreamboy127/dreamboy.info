---
layout: post
title: 推荐系统 找出内容近似的文章
categories: [nlp, recommend]
tags: [nlp, tfidf, svd, newsgroup, recommend]
description: 根据文章中每个词的TFIDF值，对文章形成对应的TFIDF向量，根据该向量的相似度，来得到相关主题的文章，进行推荐
---

本文参照[该文](http://www.52nlp.cn/category/%E6%8E%A8%E8%8D%90%E7%B3%BB%E7%BB%9F)   
本文使用java实现对`newsgroup18828`内容进行推荐（基于内容的推荐）
找出内容近似的文章，使用的特征为词的`tfidf`
算法的思想是：对语料库的`tfidf`矩阵进行`SVD`分解，可得到`V`矩阵（原矩阵为`term-doc`），根据`S`的特征值大，取前`95%`的特征值，对矩阵`S`进行降维，降维后的维度大小可视为主题数量，对`V`矩阵也进行降维操作，如`TFIDF`为`m*n`矩阵`(m>n)`，则得到`U`为`m*r`，`S`为`r*r`，`V`为`n*r`，`r`为主题数量，最后得到的`V`为`doc-topic`矩阵，可视为`n`个`r`维向量，文档相似性根据这些向量来确定；当要对一个新文档进行推荐其他文档时，先得到该文档的`TF`向量，然后归一化后与语料的`IDF`相乘，得到查询文档的`TFIDF`向量(`m`维)，对该向量进行变换，
`Q=q'*U*inv(S)`，得到的`Q`为该文档的主题向量(`r`维)，与`V`矩阵的`n`个`r`维向量进行求余弦距离，对距离进行排序，排序后的索引值对应文档就是要推荐的文档
{{ site.excerpt_separator }}
####训练过程代码如下：
```java
public void train(String saveDir) throws Exception{
	File saveDirFile=new File(saveDir); 
	if(!saveDirFile.exists())
		saveDirFile.mkdir();
	long learnStart = System.currentTimeMillis();
	corpus.computeTFIDF();
	saveSerializedFile(corpus,saveDir+"\\Corpus.obj");
	double [][] tfidfmatrix = new double[corpus.getDocSize()][corpus.getTermSize()]; 
	corpus.printTFIDF(tfidfmatrix);
	Matrix tfidfMatrix=new Matrix(tfidfmatrix);
	System.out.println("tfidfMatrix = U S V^T\nStart Decomposition\nPlease Wait...");
    s = tfidfMatrix.transpose().svd();
    S = s.getS();
    System.out.println("Sigma = "+S.getRowDimension()+"*"+S.getColumnDimension());
    double orgintrace=S.trace();
	double threshlodTrace=threshlod*orgintrace;
	int reduceDimension;
	for(reduceDimension=S.getRowDimension()-1;S.getMatrix(0, reduceDimension, 0, reduceDimension).trace()>=threshlodTrace;reduceDimension--);
	reduceDimension++;
    S=S.getMatrix(0, reduceDimension, 0, reduceDimension);
    saveSerializedFile(S,saveDir+"\\S.obj");
    System.out.println("After reduce dimension...\nSigma = "+S.getRowDimension()+"*"+S.getColumnDimension());
    U = s.getU();
    System.out.println("U = "+U.getRowDimension()+"*"+U.getColumnDimension());
    U=U.getMatrix(0, U.getRowDimension()-1, 0, reduceDimension);
    saveSerializedFile(U,saveDir+"\\U.obj");
    System.out.println("After reduce dimension...\nU = "+U.getRowDimension()+"*"+U.getColumnDimension());
    V = s.getV();
    System.out.println("V = "+V.getRowDimension()+"*"+V.getColumnDimension());
    V=V.getMatrix(0, V.getRowDimension()-1, 0, reduceDimension);
    saveSerializedFile(V,saveDir+"\\V.obj");
    System.out.println("After reduce dimension...\nV = "+V.getRowDimension()+"*"+V.getColumnDimension());
    System.out.println("rank = " + s.rank());
    System.out.println("condition number = " + s.cond());
    System.out.println("2-norm = " + s.norm2());
    long learnEnd = System.currentTimeMillis();
	System.out.println("train takes time:"+Printer.printTime(learnEnd - learnStart));
	termToindexMap=corpus.getTermToIndex();
	idf=corpus.getIdf();
	VVT=V.times(V.transpose());
}
```
####代码使用了Serializable接口，可以保存object，下次只需要使用load就可以，代码如下:
```java
public void load(String loadDir) throws Exception{
	corpus=(Corpus)readSerializedFile(loadDir+"\\Corpus.obj");
	S=(Matrix)readSerializedFile(loadDir+"\\S.obj");
	U=(Matrix)readSerializedFile(loadDir+"\\U.obj");
	V=(Matrix)readSerializedFile(loadDir+"\\V.obj");
	
	termToindexMap=corpus.getTermToIndex();
	idf=corpus.getIdf();
	VVT=V.times(V.transpose());
}
```
####查询过程代码如下：
```java
public void QueryRun(String queryDir) throws Exception{
	long queryStart = System.currentTimeMillis();
	Corpus Query = new Corpus();
	Query.readDocs(queryDir);
	double [][] QuerytfidfMatrix = new double[Query.getDocSize()][Query.getTermSize()]; 
	Query.printTF(QuerytfidfMatrix);
    double [][] QueryMatrix = new double[Query.getDocSize()][U.getRowDimension()]; 
    for(int j=0;j<Query.getDocSize();j++){
    	for(int i=0;i<Query.getTermSize();i++){
    		if(termToindexMap.containsKey(Query.getIndexToTerm().get(i)))
    		{
    			int termindex=termToindexMap.get(Query.getIndexToTerm().get(i));
    			QueryMatrix[j][termindex]=QuerytfidfMatrix[j][i]/Query.getTermSize()*idf.get(termindex);
    		}
    	}
    }
	Matrix QueryTfIdfMatrix= new Matrix(QueryMatrix);
	System.out.println("QueryTfIdfMatrix = "+QueryTfIdfMatrix.getRowDimension()+"*"+QueryTfIdfMatrix.getColumnDimension());
	//QueryTfIdfMatrix.print(9, 6);
	Matrix QueryTopicMatrix= QueryTfIdfMatrix.times(U.times(S.inverse()));
	System.out.println("QueryTopicMatrix = "+QueryTopicMatrix.getRowDimension()+"*"+QueryTopicMatrix.getColumnDimension());
	//QueryTopicMatrix.print(9, 6);
	Matrix length=new Matrix(Query.getDocSize(),V.getRowDimension());
	Matrix QueryTopicMatrixQueryTopicMatrixT=new Matrix(QueryTopicMatrix.times(QueryTopicMatrix.transpose()).getArray());
	for(int i=0;i<Query.getDocSize();i++)
		for(int j=0;j<V.getRowDimension();j++)
			length.set(i, j, Math.sqrt(VVT.get(j, j))*Math.sqrt(QueryTopicMatrixQueryTopicMatrixT.get(i, i)));
	Matrix sim = QueryTopicMatrix.times(V.transpose()).arrayRightDivideEquals(length);
	System.out.println("similarities = "+sim.getRowDimension()+"*"+sim.getColumnDimension());
	//sim.print(3, 7);
	
	double [][] ResultMatrix=sim.getArrayCopy();
	for(int i=0;i<ResultMatrix.length;i++)
	{
		/*int maxIndex=0;
		double maxSim=0.0;
		for(int j=0;j<ResultMatrix[i].length;j++)
		{
			if(maxSim<ResultMatrix[i][j]){
				maxSim=ResultMatrix[i][j];
				maxIndex=j;
			}
		}*/
		int[] index=new int[ResultMatrix[i].length];
		for(int j=0;j<index.length;j++)
			index[j]=j;
		Sort.quickSort(ResultMatrix[i], index, 0, ResultMatrix[i].length-1, false);
		for(int j=0;j<10;j++)
			System.out.println(corpus.getDocs().get(index[j]).getDocName()+":"+ResultMatrix[i][j]);
		System.out.println("\n");
	}
	long queryEnd = System.currentTimeMillis();
	System.out.println("query takes time:"+Printer.printTime(queryEnd - queryStart));
}
```

推荐过程基本如上，github:<https://github.com/dreamboy127/NLPBasicRecommend>   
如果文档数量过多，由于使用的是穷举办法，所以需要对这些向量进行建立索引，可以使用KDTREE，如果主题数量比较多，超过10个，可以使用LSH，这些属于KNN分类算法	