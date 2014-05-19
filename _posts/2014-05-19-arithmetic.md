---
layout: post
title:  四则运算栈实现
categories: [acm]
tags: [acm,数据结构]
description: 使用栈数据结构实现四则运算，加减乘除和括号
---
```c
// LinkStack.cpp : Defines the entry point for the console application.
//
/*作者:dreamboy*/
/*日期:2012.4.21*/
/*功能:四则运算*/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define DATA 1
#define OPER 0
#define OVER -1
{{ site.excerpt_separator }}
typedef struct StackNode
{
	int data;
	struct StackNode* next;
}StackNode,*LinkStackPtr;
typedef struct LinkStack
{
	LinkStackPtr top;
	int count;
}LinkStack;
typedef struct DataorOper
{
	int data;
	int dataoroper;
}DataorOper;


int Compute(LinkStack *S,const DataorOper* backString);//通过后缀表达式计算结果
int StackEmpty(LinkStack S);//检测堆栈是否空
int Push(LinkStack *S,int e);//压栈
int Pop(LinkStack *S,int *e);//弹栈
int operate(char oper,int operatedata1,int operatedata2);//运算操作
void frontString2backString(LinkStack *L,const DataorOper* S,DataorOper *D);//中缀表达式转化为后缀表达式
char checkprior(char operator1,char operator2);//比较运算符优先级
void String2frontString(char *String,DataorOper *D);//字符串转换为中缀表达式

char operators[7]="+-*/()";//运算符数组
char prioritytable[6][6]=  //运算符优先级数组
{
	{'<','<','<','<','>','o'},
	{'<','<','<','<','>','o'},
	{'>','>','<','<','>','o'},
	{'>','>','<','<','>','o'},
	{'>','>','>','>','>','o'},
	{'<','<','<','<','=','o'},
};

int main(int argc, char* argv[])
{
	char String[30];
	DataorOper frontString[30];
	DataorOper backString[30];
	LinkStack *L=(LinkStack*)malloc(sizeof(LinkStack));
	L->count=0;
	L->top=NULL;
	printf("input please!\n");
	scanf("%s",String);
	String2frontString(String,frontString);
	frontString2backString(L,frontString,backString);
	printf("%d\n",Compute(L,backString));
	return 0;
}
int Push(LinkStack *S,int e)
{
	LinkStackPtr s=(LinkStackPtr)malloc(sizeof(StackNode));
	s->data=e;
	s->next=S->top;
	S->top=s;
	S->count++;
	return 1;
}
int Pop(LinkStack *S,int *e)
{
	LinkStackPtr p;
	if(StackEmpty(*S))
		return 0;
	*e=S->top->data;
	p=S->top;
	S->top=S->top->next;
	free(p);
	S->count--;
	return 1;
}
int Gettop(LinkStack *S,int *e)
{
	if(StackEmpty(*S))
		return 0;
	*e=S->top->data;
	return 1;
}
int StackEmpty(LinkStack S)
{
	if(S.count==0)
	return 1;
	else 
	return 0;
}
int operate(char oper,int operatedata1,int operatedata2)
{
	int temp;
	switch(oper)
	{
	case '+':temp=operatedata1+operatedata2;break;
	case '-':temp=operatedata1-operatedata2;break;
	case '*':temp=operatedata1*operatedata2;break;
	case '/':temp=operatedata1/operatedata2;break;
	default:break;
	}
	return temp;
}
int Compute(LinkStack *S,const DataorOper* backString)
{
	int temp1,temp2,temp;
	while(backString->dataoroper!=OVER)
	{
		if(backString->dataoroper==DATA)
			Push(S,backString->data);
		else if(backString->dataoroper==OPER)
		{
			Pop(S,&temp1);
			Pop(S,&temp2);
			Push(S,operate((char)backString->data,temp2,temp1));
		}
		backString++;
	}
	Pop(S,&temp);
	return temp;
}
void frontString2backString(LinkStack *L,const DataorOper* S,DataorOper *D)
{
	int temp,flag,flagPush;
	while(S->dataoroper!=OVER)
	{
		flag=1;
		flagPush=0;
		if(S->dataoroper==OPER)
		{
			while(flag==1&&StackEmpty(*L)==0)
			{
				Gettop(L,&temp);
				switch(checkprior(S->data,temp))
				{
				case '>':	Push(L,S->data);flag=0;break;
				case '=':   Pop(L,&temp);
								flag=0;
								break;
				case '<':	Pop(L,&temp);
								D->data=temp;
								D->dataoroper=OPER;
								D++;
								break;
				default:break;
				}
			}
			if(StackEmpty(*L))
			{
				Push(L,S->data);
			}
		}
		else if(S->dataoroper==DATA)
		{	
			D->data=S->data;
			D->dataoroper=DATA;
			D++;
		}
		S++;
	}
	while(StackEmpty(*L)==0)
	{
	Pop(L,&temp);
	D->data=temp;
	D->dataoroper=OPER;
	D++;
	}
	D->data=-1;
	D->dataoroper=OVER;
}
char checkprior(char operator1,char operator2)
{
	int i,j;
	i=strchr(operators,operator1)-operators;
	j=strchr(operators,operator2)-operators;
	return prioritytable[i][j];
}
void String2frontString(char *String,DataorOper *D)
{
	int sum;
	while(*String!='\0')
	{
		if(*String>='0'&&*String<='9')
		{
			sum=0;
			while(*String>='0'&&*String<='9')
			{
				sum=sum*10+*String-'0';
				String++;
			}
			D->data=sum;
			D->dataoroper=DATA;
			D++;
		}
		else
		{
			D->data=*String;
			D->dataoroper=OPER;
			D++;
			String++;
		}
	}
	D->data=-1;
	D->dataoroper=OVER;
}
```


