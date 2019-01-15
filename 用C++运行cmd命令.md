## 学习C++

用C++运行cmd命令,设置git的http代理,减少手动输入的次数
```
// gitset.cpp: 定义控制台应用程序的入口点。
//

#include "stdafx.h"
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
/*
#include <cstdio>
#include <iostream>
using namespace std;
*/

std::string httpProxy = "git config --global https.proxy https://192.168.1.10:8888";
std::string httpsProxy = "git config --global http.proxy http://192.168.1.10:8888";
std::string clearHttp = "git config --global --unset http.proxy";
std::string clearHttps = "git config --global --unset https.proxy";
std::string showConfig = "git config --list";

void print_result(FILE *fp)
{
	char buf[100];

	if (!fp)
	{
		return;
	}
	printf("\n>>>\n");
	while (memset(buf, 0, sizeof(buf)), fgets(buf, sizeof(buf) - 1, fp) != 0) {
		printf("%s", buf);
	}
	printf("\n<<<\n");
}

void gitHelp() {
	FILE *fp = NULL;
	fp = _popen("git --help", "r");

	if (fp != NULL) {
		print_result(fp);
	}
	_pclose(fp);
}

void proxyRun(std::string cmd) {
	FILE *fp = NULL;
	fp = _popen(cmd.c_str(), "r");

	if (fp != NULL) {
		print_result(fp);
	}
	_pclose(fp);
}

int main(int argc, char* argv[])
{
	if (argc != 2) {
		std::cout << "gitset.exe  set 设置代理 " << std::endl;
		std::cout << "gitset.exe  set 取消代理" << std::endl;
		return 0;
	}

	for (int i = 0; i < argc; i++) {
		std::cout << argv[i] << std::endl;
	}

	char* param = argv[1];
	int isSet = strcmp(param, "set");
	if (isSet == 0) {
		std::cout << "do set proxy" << std::endl;

		proxyRun(httpProxy);
		proxyRun(httpsProxy);
	}
	int isUnset = strcmp(param, "unset");
	if (isUnset == 0) {
		std::cout << "do unset proxy" << std::endl;
		proxyRun(clearHttp);
		proxyRun(clearHttps);
	}

	proxyRun(showConfig);
	system("pause");
    return 0;
}



```
