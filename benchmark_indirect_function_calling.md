# Benchmark: Indirect Function Calling

I tested calling a function directly, via a function call and via a lambda.
```C++
std::function<int(int,int)> func_lambda;
int (*func_ptr)(int,int);

int mul(int a,int b){return a*b;}

int main(int argc, char *argv[])
{
    func_lambda=mul;
    func_ptr=mul;

    for(int i=0;i<5;i++)
    {
        int s=0;
        timer _("direct");
        for(int i=0;i<100000000;i++)
            s+=mul(i,i);
        std::cout<<s<<'\t';
    }
    for(int i=0;i<5;i++)
    {
        int s=0;
        timer _("via function pointer");
        for(int i=0;i<100000000;i++)
            s+=func_ptr(i,i);
        std::cout<<s<<'\t';
    }
    for(int i=0;i<5;i++)
    {
        int s=0;
        timer _("via lambda");
        for(int i=0;i<100000000;i++)
            s+=func_lambda(i,i);
        std::cout<<s<<'\t';
    }
    for(int i=0;i<5;i++)
    {
        int s=0;
        timer _("lambda direct 1");
        for(int i=0;i<100000000;i++)
            s+=[](int a,int b){return a*b;}(i,i);
        std::cout<<s<<'\t';
    }
    for(int i=0;i<5;i++)
    {
        auto lambda=[](int a,int b){return a*b;};
        int s=0;
        timer _("lambda direct 2");
        for(int i=0;i<100000000;i++)
            s+=lambda(i,i);
        std::cout<<s<<'\t';
    }
    return 0;
}
```

Result (build with -O2 with GCC 4.8.1):
```
-1452071552    0.057003 	<- direct
-1452071552    0.058003 	<- direct
-1452071552    0.058003 	<- direct
-1452071552    0.058003 	<- direct
-1452071552    0.058003 	<- direct
-1452071552    0.17501 	<- via function pointer
-1452071552    0.18101 	<- via function pointer
-1452071552    0.18101 	<- via function pointer
-1452071552    0.17701 	<- via function pointer
-1452071552    0.174009 	<- via function pointer
-1452071552    0.267015 	<- via lambda
-1452071552    0.263015 	<- via lambda
-1452071552    0.261015 	<- via lambda
-1452071552    0.262014 	<- via lambda
-1452071552    0.259014 	<- via lambda
-1452071552    0.058003 	<- lambda direct 1
-1452071552    0.057003 	<- lambda direct 1
-1452071552    0.057003 	<- lambda direct 1
-1452071552    0.058003 	<- lambda direct 1
-1452071552    0.057003 	<- lambda direct 1
-1452071552    0.058003 	<- lambda direct 2
-1452071552    0.057003 	<- lambda direct 2
-1452071552    0.057003 	<- lambda direct 2
-1452071552    0.059003 	<- lambda direct 2
-1452071552    0.059003 	<- lambda direct 2
```
