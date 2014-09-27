#define _SVID_SOURCE
#include<stdio.h>
#include<sys/types.h>
#include<string.h>
#include<sys/wait.h>
#include<signal.h>
#include<stdlib.h>
#include<fcntl.h>
#include<ctype.h>
#include<unistd.h>
#include<setjmp.h>
#include<math.h>
#define MAX 150
#define BUFSIZE 256
char last_dir[MAX];
int divflag=0;
char path[MAX];
char prompt[MAX];
char homedir[MAX];
char cwd[1024];
int result;
jmp_buf(getinput);
   int x,y;
   char* var1;
   char *xvalue, *yvalue;
   char *xtoken, *ytoken, *xvar, *xval, *yvar, *yval;
   char *xx,*yy;

int power(int x, int y)
{
    if( y == 0)
        return 1;
    else if (y%2 == 0)
        return power(x, y/2)*power(x, y/2);
    else
        return x*power(x, y/2)*power(x, y/2);
 
}

/*lock file*/
int readLock()
{
    FILE *file = fopen("/lock","r");
    if(!file)
    {
        FILE *file = fopen("/lock","a");
        return 0;
    }
    else
        return 1;
}


/* Reading Profile file */
int readProfile()
{

	printf("Reading Profile.txt and setting environment variables..\n");
	FILE *file = fopen("profile.txt","r");
	if (!file) {
        fprintf(stderr, "Cannot open file 'profile.txt'!\n");
        return 1;
    }
	int i=0;
	int j=0;
	char parse[MAX];
	char disp[MAX];
	char s;
	do
	{
		fscanf(file,"%c",&s);
	}while(s!='-');
	do
	{
		fscanf(file,"%c",&s);
		path[j]=s;
		j++;
	}while(s!='\n');
		path[j-1]='\0';
		printf("\npath:%s",path);
		j=0;
	do
	{
		fscanf(file,"%c",&s);
	}while(s!='-');
	do
	{
		fscanf(file,"%c",&s);
		homedir[j]=s;
		j++;
	}while(s!='\n');
		homedir[j-1]='\0';
		printf("\nhomedir:%s",homedir);
		return 0;
	}


/*set the default profile*/
int defaultProfile()
{
    FILE *file = fopen("profile.txt","w+");
    fprintf(file,"path -/bin:/usr/bin\nhome -/root\n");
    fclose(file);
    return 0;
}

/* setting home directory */
int sethomedir(char homedir[])
{
 int k;
 k=chdir(homedir);
 if(k!=0)
{
  fprintf(stderr,"\nError,Home Directory not set..\n");
  return 0;
}
else
printf("\nHome directory is set");
return 1;
}


/* Displaying current working directory */
int displaycurrdir()
{

  
  if(getcwd(cwd,sizeof(cwd)) != NULL)
    printf("\nCurrent Working Directory is - %s\n",cwd);
  else{
    fprintf(stderr,"\n Error in printing current working dir\n");
    return 1;
      } 
return 0;
}


/*Setting the environment path */
void setenv_path()
{
  char var[100];
  strcpy(var,"PATH=");
  strcat(var,path);
  //putenv(var);
 setenv("PATH", path, 1);
  printf("\nPATH SET IS - %s\n",getenv("PATH"));

}

/* Handling CTRL+C function */
void ctrlc_handler(int f)
{

char alter[150];  
int st;
  printf("\n Exit from the shell: Are you sure? [yes/anykey to abort]: ");
  scanf("%31s", alter);

  if((strcmp(alter, "y") == 0) || (strcmp(alter, "Y") == 0) || (strcmp(alter, "yes") == 0)
      || (strcmp(alter, "YES") == 0))
  {
      st=remove("/lock");
	  //if(st==0) printf("lock file deleted\n");
				//else printf("not able to remove lock file\n");
    exit(0);
  }
  else 
   {
    signal(SIGINT,ctrlc_handler);
    longjmp(getinput,1);
   }
 


}


/*Redirect Command for ">" "=>" */
int redirectCommand(char* line) {
	FILE *pipein_fp, *redirect_fp;
	int redirect;
	char readbuf[128];
	int i, j;
	char* command1;
	char* command2;
	int pid, fd;
	int flag = 1;
	char delimiters[]="=>>";
	command1 = strtok(line, delimiters);
	command2 = strtok(NULL, delimiters);
        if(command2==NULL){
        printf("\n Enter the file to redirect");
        return 0;
	}
	redirect_fp = fopen(command2, "w");
	if ((pipein_fp = popen(command1, "r")) == NULL) {
		longjmp(getinput, 1);
	}
	while (fgets(readbuf, 80, pipein_fp)) {
	flag=0;
		fputs(readbuf, redirect_fp);
		fflush(redirect_fp);
	}
	fclose(redirect_fp);
	pclose(pipein_fp);
	return flag;
}


/* Calculator operation */
int calculate(char* line1){

  
   char str[32];
   char str1[32];
   char res[32];
   int var;
   int result1;
   char *resval;
   char *token,*token1, *token2;
   xvalue=getenv("XVAR");
   yvalue=getenv("YVAR");
   char delimiters[]=" ()\r\n";
   token=strtok(line1,delimiters);
   token1=strtok(NULL,delimiters);
   token2=strtok(NULL,delimiters);
   int input[250];
   int i=0;
   int flag=1;
   x = atoi(xvalue);    // for converting char* to int
   y = atoi(yvalue);
 
  
    if(strstr(line1,"add") != NULL){
     result = x+y;
    }
  
    if(strstr(line1,"subtract") != NULL){
     result = x-y;  
    }
    if(strstr(line1,"multiply") != NULL){
     result = x*y;
    }
    if(strstr(line1,"divide") != NULL){
      if(y!=0)
      result = x/y;
      else{
      divflag=1;
      return 0;
      } 
    }
    if(strstr(line1,"power") != NULL){
      result=power(x,y);
    }
     snprintf(res, sizeof(res), "%d", result);
    setenv("result",res,1);               // similar function as 'export name=value' in bash
    system("echo result=$result") ;       // executing echo command in C script
   return 0;
}

/* Function to execute cd command */
int executeCdCommand(char command[]) {
	int chk;
	char dir[100];
	char last_dir_temp[MAX];
	int i = 0, j = 0;
	while ((command[i] != ' ') && (command[i] != '\0'))
		i++;
	if (command[i] == ' ') {
		i++;
		while (command[i] != '\0') {
			dir[j++] = command[i++];
		}
		dir[j] = '\0';

		if (dir[0] == '-') {
			getcwd(last_dir_temp, MAX);
			chdir(last_dir);
			strcpy(last_dir, last_dir_temp);
		} else {
			getcwd(last_dir_temp, MAX);
			chk = chdir(dir);
			if (chk != 0) {
				printf("\nError Setting directory\n");
				return 0;
			}
			strcpy(last_dir, last_dir_temp);
			return 1;
		}
	} else if (command[i] != ' ') {
		getcwd(last_dir, MAX);
		sethomedir(homedir);
		return 1;
	}
	return 0;
}

/* Function to execute normal commands without arguments like ls,cat etc. */

int executeCommand(char* line) {

	
	int flag = 1;
	FILE *pipein_fp;
	char readbuf[80];
	if ((pipein_fp = popen(line, "r")) == NULL) {
		printf("Error executing command\n");
		fprintf(stderr, "COMMAND NOT FOUND\n");
		perror("popen");
		longjmp(getinput, 1);
	}
	while (fgets(readbuf, 80, pipein_fp))
	{
		flag=0;
		printf("%s", readbuf);
	}
	

	pclose(pipein_fp);
	return flag;

}


/* Parsing input command for pipe and nesting */
int split_pipes(char *buf, char **args, int max)
{
    int n=0;
    char *arg=strchr(buf, '#');
    char delimiters[]="|()$\r\n";
	
    // Chop off comments from end, if any
    if(arg != NULL)    arg[0]='\0';

    // Cut apart based on pipes.  Also eat \r and \n from end of line
    arg=strtok(buf,delimiters);
    while(arg != NULL)
    {
        args[n++]=arg;
        arg=strtok(NULL,delimiters);
        if(n >= max) break;
    }
    if(n<2)
	  {
        printf("\n Enter atleast one file to pipe with");
        return 0;
	}
    return(n);
}

/* Function to process multiple pipe commands and multiple Nesting commands */
pid_t create_process(char *part, int const pipes[][2], int pipenum)
{
    pid_t pid;
    char *args[64];
    int argc=0, n;
    char *arg=strtok(part, " \t");

    while(arg != NULL)
    {
        args[argc++]=arg;
        arg=strtok(NULL, " \t");
    }

    args[argc++]=NULL;

    for(n=0; args[n] != NULL; n++)
    {
       // fprintf(stderr, "\t\targ %02d: %s\n", n, args[n]);
    }

    pid = fork();
    if(pid==-1)
	{
	perror("Could not fork\n");
	return -1;
	}
	if(pid == 0)
    {
        int m;

        if(pipes[pipenum][STDIN_FILENO] >= 0) dup2(pipes[pipenum][STDIN_FILENO], STDIN_FILENO); // FD 0
        if(pipes[pipenum][STDOUT_FILENO] >= 0) dup2(pipes[pipenum][STDOUT_FILENO], STDOUT_FILENO); // FD 1

        // Close all pipes
        for(m=0; m<64; m++)
        {
            if(pipes[m][STDIN_FILENO] >= 0) close(pipes[m][STDIN_FILENO]);
			
            if(pipes[m][STDOUT_FILENO] >= 0) close(pipes[m][STDOUT_FILENO]);
        }

        execvp(args[0], args);
        fprintf(stderr, "COMMAND NOT FOUND\n");
        exit(255);
    }
	

    return(pid);
}


void pipeCompute(char line1[])
{
	int n,m;
	pid_t children[64];
    int pipes[64][2];
    char *parts[64];
    char buf[512];
    int linenum=0;
	char *argv[BUFSIZE];
    n=split_pipes(line1, parts, 64); // Split apart line into 'parts'

        // Clear out any pipes from last loop
        for(m=0; m<64; m++)
        {
          pipes[m][STDIN_FILENO]=-1; pipes[m][STDOUT_FILENO]=-1;
        }

        // For n processes, create n-1 pipes
        for(m=0; m<(n-1); m++)
        {
            int p[2];
            pipe(p);
            pipes[m][STDOUT_FILENO]=p[STDOUT_FILENO]; // Process N writes to the pipe
            pipes[m+1][STDIN_FILENO]=p[STDIN_FILENO]; // Process N+1 reads from the pipe
        }

        for(m=0; m<n; m++)
        {
            //fprintf(stderr, "\tpart %02d: %s\n", m, parts[m]);
            children[m]=create_process(parts[m], pipes, m);
        }

        // Close all pipes
        for(m=0; m<64; m++)
        {
            if(pipes[m][STDIN_FILENO] >= 0) close(pipes[m][STDIN_FILENO]);
            if(pipes[m][STDOUT_FILENO] >= 0) close(pipes[m][STDOUT_FILENO]);
        }

        for(m=0; m<n; m++)
        {
                int stat;

                waitpid(children[m], &stat, 0);
                //fprintf(stderr, "Child %d returned %d\n",m, WEXITSTATUS(stat));
        }
  
}
void setvarFn(char *line1)
{

char delimiters[]=" =\r\n";

int value;
if(strstr(line1, "setvar")!=NULL && strstr(line1, "x")!=NULL){
xtoken=strtok(line1,delimiters);
xvar=strtok(NULL,delimiters);
xval=strtok(NULL,delimiters);
 setenv("XVAR",xval,1);
 xx=getenv("XVAR");
 if(sscanf(xx, "%d", &value) != 1){
  printf("\n Please enter an integer for x");
   
   } 

}
else if(strstr(line1, "setvar")!=NULL && strstr(line1, "y")!=NULL){
ytoken=strtok(line1,delimiters);
yvar=strtok(NULL,delimiters);
yval=strtok(NULL,delimiters);
 setenv("YVAR",yval,1);
yy=getenv("YVAR");
 if(sscanf(yy, "%d", &value) != 1){
  printf("\n Please enter an integer for y");
   
   } 
}
 
}


/****main function ****/

int main(int argc,const char **argsv)
{

       /*if(readLock()!=0)
       {exit(0);}  */
	//printf("\n\n*****My New Shell****\n");
	printf("Supriya A20320511\n");
	printf("Sruthi A20319996\n");
	signal(SIGINT,ctrlc_handler);
    if(readProfile()!=0)
    {
        defaultProfile();
        readProfile();
    }
	printf("\nEnvironmental variables set !!\n ");
	int flag1=sethomedir(homedir);
	char str[MAX];
	int result;

	int k=displaycurrdir(); 
	setenv_path();
        setjmp(getinput);
	/* getting user input */
	char* line=NULL;
	char line1[100];
	size_t len=0;
	ssize_t read;
	printf("\n%s$ ",cwd);
    int st;
	int flag=0;
	while((read=getline(&line,&len,stdin))!=-1)
	{
		if(strlen(line)>0 && line[strlen(line)-1]=='\n')
			line[strlen(line)-1]='\0';
			strcpy(line1,line);

		if(strcmp(line1,"exit")==0)
		{
			printf("\n ....Exiting.....\n");
            
                st=remove("/lock");
				//if(st==0) printf("lock deleted\n");
				//else printf("lock is not able to remove\n");
                exit(0);
            

		}

		if (strstr(line1, "=>") != NULL||strstr(line1, ">") != NULL) {
   			redirectCommand(line1);
		 }
              
        else if (strstr(line1, "(") != NULL||strstr(line1, "|") != NULL) {       
		          pipeCompute(line1);
			
		 }
		else if ((line1[0]=='c')&&(line1[1]=='d')&&((line1[2]==' ')||(line1[2]=='\0')))
            {
                executeCdCommand(line1);
            }
	    else if (strstr(line1, "setvar") != NULL)
		 {

		setvarFn(line1);
		flag=1;
		}
		else if (strstr(line1,"add")||strstr(line1,"subtract") != NULL||strstr(line1,"multiply") != NULL||strstr(line1,"divide") != NULL||strstr(line1,"power") != NULL) 
	      
		 {
		//if(flag==1){
		
		int res=calculate(line1);
		if(divflag==1)
              printf("\nDivision by zero not allowed");
            
         
			//}
		/*else
		printf("\nPlease set variables first");*/
		}


        else 
            executeCommand(line1);
                  
	divflag=0;
	printf("\n%s$ ",cwd);
  }
    
    
	return 0;
}

