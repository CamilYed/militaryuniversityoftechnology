

# Projekt zaliczeniowy z systemów operacyjnych.

## Opracować zestaw programów typu producent-konsument realizujących przy wykorzystaniu mechanizmu semaforów i pamięci dzielonej, następujący schemat komunikacji międzyprocesowej: 

* Proces 1: czyta dane (pojedyncze wiersze) ze standardowego strumienia wejściowego i przekazuje je 
w niezmienionej formie do procesu 2.
* Proces 2: pobiera dane przesłane przez proces 1. Oblicza ilość znaków w każdej linii i wyznaczoną
liczbę przekazuje do procesu 3. 
* Proces 3: pobiera dane wyprodukowane przez proces 2 i umieszcza je w standardowym strumieniu 
wyjściowym. Każda odebrana jednostka danych powinna zostać wyprowadzona w osob
nym wierszu. 

### Należy zaproponować i zaimplementować mechanizm informowania się procesów o swoim stanie. 

Należy wykorzystać do tego dostępny mechanizm sygnałów i kolejek komunikatów. 
Wszystkie trzy procesy powinny być powoływane automatycznie z jednego procesu inicjującego. 
Należy zaimplementować mechanizm asynchronicznego przekazywania informacji pomiędzy operatorem a procesami 
oraz pomiędzy procesami. Wykorzystać do tego dostępny mechanizm sygnałów i kolejek komunikatów. 

Operator może wysłać do dowolnego procesu sygnał zakończenia działania (S1), sygnał wstrzymania działania (S2) i 
sygnał wznowienia działania (S3). Sygnał S2 powoduje wstrzymanie synchronicznej wymiany danych pomiędzy procesami. 
Sygnał S3 powoduje wznowienie tej wymiany. Sygnał S1 powoduje zakończenie działania oraz zwolnienie wszelkich wykorzy-
stywanych przez procesy zasobów. 

Każdy z sygnałów przekazywany jest przez operatora tylko do jednego, dowolnego procesu. O tym, do którego proce-
su wysłać sygnał, decyduje operator, a nie programista. Każdy z sygnałów operator może wysłać do innego procesu. Mimo, że 
operator kieruje sygnał do jednego procesu, to pożądane przez operatora działanie musi zostać zrealizowane przez wszystkie 
trzy procesy. W związku z tym, proces odbierający sygnał od operatora musi powiadomić o przyjętym żądaniu pozostałe dwa 
procesy. Powinien wobec tego wysłać do nich sygnał (S4) oraz przekazać informację o tym jakiego działania wymaga operator, 
przekazując im stosowny komunikat (lub komunikaty) poprzez mechanizm sygnałów i kolejek komunikatów.
Procesy odbierające sygnał S4, powinny odczytać skierowany do nich komunikat (lub komunikaty) w procedurze odbierania sy-
gnału S4.

Wszystkie trzy procesy powinny zareagować zgodnie z żądaniem operatora. 
Sygnały oznaczone w opisie zadania symbolami S1 ÷ S4 należy wybrać samodzielnie spośród dostępnych w systemie (np. 
SIGUSR1, SIGUSR2, SIGINT, SIGCONT).

UWAGA: 
Wszelkie wątpliwości związane z treścią zadania należy wyjaśniać z prowadzącym zajęcia laboratoryjne.
```c

 #include <sys/types.h>
 #include <sys/ipc.h>
 #include <sys/shm.h>
 #include <sys/sem.h>
 #include <signal.h>
 #include <unistd.h>
 #include <stdio.h>
 #include <stdlib.h>
 #include <errno.h>  
 #include <sys/msg.h>
 #include "err.h"
 #include <fcntl.h>
 #include <string.h>
 
 #define SIZE_OF_SEG1 1024
 #define SIZE_OF_SEG2 5
 #define SIZE_BUF1 1024
 
 #define PAUSE 0
 #define CONT  1
 #define KILL  2
 #define END_READ 3

 
 key_t key;                                                         // Klucz dla semaforów
 key_t key_s1;                                                      // Klucz dla segmentu pamięci współdzielonej 1
 key_t key_s2;                                                      // Klucz dla segmentu pamięci współdzielonej 2
 key_t key_q;                                                       // Klucz dla kolejki komunikatów
 static int pid[3];                                                 // Tablica pidów procesów
 int start_stop = 1;                                                // Flaga startu i stopu
 int queue;                                                         // Identyfikator kolejki komunikatow
 char BUF1[SIZE_BUF1];                                              // Bufor do zczytywania linii
 int semid;                                                         // Identyfikator zestawu semaforów
 int shmid1,shmid2;                                                 // Identyfikatory segmentów pamięci
 char *buf = NULL;                                                  // Wskaźnik na segment pamieci 1
 char *buf2 = NULL;                                                 // Wskaźnik na segment pamięci 2
 FILE * fp;                                                         // Wskaźnik plikowy zapis/odczyt pidow
 
 struct sembuf buff;
 
 typedef struct{ 
	long type;
	int  sig;
 } MSG;

 MSG msgreceive;
 MSG msgsend;
 
 /* Nagłówki funkcji                                        */
 void tworz_klucz(void);
 void proces1(void);
 void proces2(void);
 void proces3(void);
 void sem_operationV(int semid, int semnum);
 void sem_operationP(int semid, int semnum);
 void init_sem(void);
 void init_shared_mem(void);
 void initQueue(void);
 void sendToQueue(int signo);
 void receiveFromQueue(int signo);
 void error(char *message);
 void clean(int pid_p);
 void clean_and_finish(int signo);
 void savePID(void);
 
 /*--------------------------------------- MAIN ----------------------------------------------------------*/
 int main(int argc, char* argv[]){
 
 sigset_t mask;	
 sigemptyset(&mask);	
 
 
 init_sem();
 init_shared_mem();
 initQueue();
 /* Tworzenie pliku z pidami */
 remove("/tmp/pid");
 fp = fopen("/tmp/pid","w");
 if(fp==NULL) error("nie mozna otworzyc pliku /tmp/pid");
 /* Ustawienie obslugi sygnalow  */
 signal(SIGINT, sendToQueue);   //Zatrzymanie
 signal(SIGCONT, sendToQueue);  //Wznowienie
 signal(SIGTERM, sendToQueue);  //Zakonczenie
 
 signal(SIGUSR1, receiveFromQueue); // Do odebrania komunikato z kolejki
 signal(SIGUSR2, clean_and_finish); // Do sprzatania
 signal(SIGCHLD,SIG_IGN);

 
 
 if ((pid[2]=fork()) == 0)
 {
    proces3();
 }else if ((pid[1]=fork()) == 0)
 {
        proces2();
  }else if ((pid[0]=fork()) == 0)
  {
        proces1();	
  }
 fwrite(pid,sizeof(int),sizeof(pid),fp);
 fclose(fp);
 
 printf("Pid %d\n",pid[0]);
 printf("Pid %d\n",pid[1]);
 printf("Pid %d\n",pid[2]);
 
 sigfillset(&mask);
 sigprocmask(SIG_BLOCK,&mask,NULL);
 
 wait();
 return EXIT_SUCCESS;
 }
 /*--------------------------------------- END MAIN ------------------------------------------------------*/
 
 
 /* Funkcja tworzy unikalny klucz                           */
 void create_key(void){
    
    if ((key = ftok(".", 1)) == -1){
        fprintf(stderr, "blad tworzenia klucza\\n");
        exit(EXIT_FAILURE);
        }else{
            printf("\tUtworzono klucz :   %d\n",key);
            
     }
    
    
    if ((key_s1 = ftok(".", 2)) == -1){
        fprintf(stderr, "blad tworzenia klucza\\n");
        exit(EXIT_FAILURE);
        }else{
            printf("\tUtworzono klucz :   %d\n",key_s1);
    }
    
    if ((key_s2 = ftok(".", 3)) == -1){
        fprintf(stderr, "blad tworzenia klucza\\n");
        exit(EXIT_FAILURE);
        }else{
            printf("\tUtworzono klucz :   %d\n",key_s2);
    }
    
    if ((key_q = ftok(".", 4)) == -1){
        fprintf(stderr, "blad tworzenia klucza\\n");
        exit(EXIT_FAILURE);
        }else{
            printf("\tUtworzono klucz :   %d\n",key_s2);
    }
    
        
 }
/* Funkcje operujące na semaforach                          */
 void sem_operationV(int semid, int semnum){
    int r;
	buff.sem_num = semnum;
    buff.sem_op = 1;
    buff.sem_flg = 0;
    /*
        Robimy tak aby ustrzec sie bledu ze operacja na semaforze zostanie przerwana przez nachodzacy sygnal
    */
    do {
        r = semop(semid, &buff, 1);
        
    }while(r < 0 && errno == EINTR);
    if (r < 0 && errno != -1){
         perror("Podnoszenie semafora");
        exit(EXIT_FAILURE);
    }
    /*
        if (semop(semid, &buff, 1) == -1){
            perror("Podnoszenie semafora");
            exit(EXIT_FAILURE);
        }
        */
 }

 void sem_operationP(int semid, int semnum){
    int r;
    buff.sem_num = semnum;
    buff.sem_op = -1;
    buff.sem_flg = 0;
    
    do {
        r = semop(semid, &buff, 1);
        
    }while(r < 0 && errno == EINTR);
    if (r < 0 && errno != -1){
        perror("Opuszczanie semafora");
        exit(EXIT_FAILURE);
    }
    
    /*
        if (semop(semid, &buff, 1) == -1){
            perror("Opuszczenie semafora");
            exit(EXIT_FAILURE);
        }
        
        */
 }
 
 void init_sem(void){
    create_key();
    // Tworzymy zestaw 3 semaforów o kluczu key
    semid=semget(key,3,IPC_CREAT|0600);				      		                    
	if(semid == -1){       
        perror("Utworzenie tablicy semaforow"); 
	    exit(EXIT_FAILURE);    
    }
    //Nadanie wartosci kazdemu z semaforow
    /*
        Semafor 0 dla Procesu nr 1 wartość 1
        Semafor 1 dla Procesu nr 2 wartość 0
        Semafor 2 dla Procesu nr 3 wartość 0
    */
    if (semctl(semid, 0, SETVAL, (int)1) == -1){ 		      
        perror("Nadanie wartosci semaforowi 0");
        exit(EXIT_FAILURE);
    }
    if (semctl(semid, 1, SETVAL, (int)0) == -1){ 		     
        perror("Nadanie wartosci semaforowi 1");
        exit(EXIT_FAILURE);
    }
    if (semctl(semid, 2, SETVAL, (int)0) == -1){ 		     
        perror("Nadanie wartosci semaforowi 2");
        exit(EXIT_FAILURE);
    }   
    
 
 }
 
 void init_shared_mem(void){
 
    shmid1=shmget(key,SIZE_OF_SEG1,IPC_CREAT|0666);        		      
    shmid2=shmget(key_s2,SIZE_OF_SEG2,IPC_CREAT|0666);        		      		
	if (shmid1 == -1){
	 
		error("Utworzenie segmentu pamięci wspołdzielonej");
		//exit(EXIT_FAILURE);
    }
    
   
    
    shmid2=shmget(key_s2,SIZE_OF_SEG2,IPC_CREAT|0666);        		      		
	if (shmid2==-1){
	    error("Utworzenie segmentu pamięci wspołdzielonej");
	}
		
		//exit(EXIT_FAILURE);
    
    
    if (shmid1 == -1 || shmid2 == -1){
        exit(EXIT_FAILURE);
    }
    
 
 }
 
 void initQueue(void){
    queue = msgget(key_q,IPC_CREAT | 0660); 
    if (queue == -1){
        error("Tworzenie kolejki");
        
        exit(EXIT_FAILURE);
    }
 }
 
 void sendToQueue(int signo){
    int p = getpid();
	int j;
	switch(signo)
	{ 
		case SIGINT:
			if(!start_stop) return;
			start_stop =0;
			printf("proces PID %d  zostal wstrzymany\n",p);
			msgsend.sig = PAUSE;
			break;

		case SIGCONT:
			if(start_stop) return;
			start_stop = 1;
			printf("proces PID %d zostal wznowiony\n",p);
			msgsend.sig = CONT;
			break;

		case SIGTERM:
			printf("proces PID %d zostanie zakonczony\n",p);
			msgsend.sig = KILL;
			break;
		
		
	}

	msgsend.type=1;
	int send1 = msgsnd(queue,&msgsend,sizeof(int),IPC_NOWAIT); 
	int send2 = msgsnd(queue,&msgsend,sizeof(int),IPC_NOWAIT); 
	if(send1==-1 ||send2==-1)
	{
		if(errno == EINTR || errno == EAGAIN)
		error("blad wyslania komunikatow");
	}
	for(j=0;j<3;j++)
		if(pid[j]!=p)
		{
		    printf("Wysylam do proces %d \n",pid[j]);
			kill(pid[j],SIGUSR1); 
		}

	if(msgsend.sig == KILL)
	{ 
		clean(p);
		exit(0);
	}
	

 
 }
 
  void receiveFromQueue(int signo){
    int p = getpid();
	int readed= msgrcv(queue,&msgreceive,sizeof(int),0,0); 
			if(readed==-1)
			{
				if(errno= EINVAL)msgreceive.sig=2;
				else 
				if(errno != ENOMSG && errno != EINTR) 
				error("blad odebrania komunikatu");
				
			}
    printf("Proces: %d odebral wiadomosc\n",p);
	switch(msgreceive.sig)
	{
		case PAUSE:
			if(!start_stop) return;
			start_stop = 0;
			printf("proces PID %d rowniez zostal wstrzymany\n",p);
			break;

		case CONT:
			if(start_stop) return;
			start_stop =1;
			printf("proces PID %d rowniez zostal wznowiony\n",p);
			break;

		case KILL:
			printf("proces PID %d rowniez zostanie zakonczony\n",p);
			clean(p);
			exit(0);
			break;

		
		
		
	}
  }
 
 /* ------------------------------- PROCESY ----------------------------------------------------------------- */
 void proces1(void){
    savePID();
    // Przyłączenie segmentu
    buf = (char *)shmat(shmid1,NULL,0);                    		      		
	if (buf == NULL){
        error("Przylaczenie segmentu pamieci wpsoldzielonej");
        exit(EXIT_FAILURE);
    }
    
    printf("Proces 1 czyta linie :\n");
    
    while (1){
        if(start_stop){
            sem_operationP(semid,0);
            fflush(stdin);
            //printf("Enter a  message: \n");
            read(STDIN_FILENO,&BUF1,SIZE_BUF1);
            //fgets(BUF1,SIZE_BUF1,stdin);
            strcpy(buf,BUF1);
            sem_operationV(semid,1);
        }
        

    }
    
 }
 
 void proces2(void){
    savePID();
    int liczba=0;
    unsigned int l=0;
    buf = (char *)shmat(shmid1,NULL,0);                    		      		
	if (buf == NULL){
        error("Przylaczenie segmentu pamieci wpsoldzielonej");
        //exit(EXIT_FAILURE);
    }
    
    buf2 = (char *)shmat(shmid2,NULL,0);                    		      		
	if (buf2 == NULL){
        error("Przylaczenie segmentu pamieci wpsoldzielonej");
        //exit(EXIT_FAILURE);
    }
    
    int sum=1;
    while(1){
        do{
            if (start_stop != 0){
                sem_operationP(semid,1);
                sum = strlen(buf)-1;
                sprintf(buf2,"%d",sum);
                sem_operationV(semid,2);
            }
        }while(sum >= 0);
    }
    
    
    
 
 }
 
 void proces3(void){
    savePID();
    buf2 = (char *)shmat(shmid2,NULL,0);                    		      		
	if (buf2 == NULL){
        perror("Przylaczenie segmentu pamieci wpsoldzielonej");
        exit(EXIT_FAILURE);
    }
    while (1){
        if (start_stop != 0){
            sem_operationP(semid,2);
            printf("Znakow odebrano: %s\n",buf2);
            sem_operationV(semid,0);
        }
    }
    
 
 
 }
 
 void error(char *message){ 
	perror(message);
	int p = getpid();
	int j;

	for(j=0;j<3;j++)
		if(pid[j]!=p)
		{
		    printf("Wysylam do procesu %d zwolnij zasoby\n",pid[j]);
			kill(pid[j],SIGUSR2);	
		}

	clean(p);
	exit(EXIT_FAILURE);
 }
 
 void clean(int pid_p){
    printf("Proces %d usuwa zasoby\n", pid_p);
    if(pid_p==pid[2])	
	 	semctl(semid, 0, IPC_RMID,0);
	 	msgctl(queue,IPC_RMID,0);
	
    // Proces 2
	if(pid_p==pid[1]){
	    if (shmdt(buf2) == -1)
	        perror("Odlaczenie segmentu");
	    if (shmctl(shmid2, IPC_RMID, NULL) == -1)
	        perror("Usuniecie segmentu");
	}
		
    // Proces 1
	if(pid_p==pid[0]){ 
	    if (shmdt(buf) == -1)
	        perror("Odlaczenie segmentu");
	    if (shmctl(shmid1, IPC_RMID, NULL) == -1)
	        perror("Usuniecie segmentu");
	}
 
 }
 
 void clean_and_finish(int signo){
	int p = getpid();
	clean(p);
	exit(-1);
 }
 
 void savePID(){ 
	fp = fopen("/tmp/pid","r");
	int r=sizeof(int)*3;
	int readed=0;
	
	while(r>0)
	{	
		readed = fread(pid,1,sizeof(pid),fp); 
		if(readed!= EOF)
		{	
			r-=readed;	
		}		
	}

	fclose(fp);
 }
 
 




```
