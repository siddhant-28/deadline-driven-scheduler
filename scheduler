/* Standard includes. */
#include <stdint.h>
#include <stdio.h>
#include "stm32f4_discovery.h"
/* Kernel includes. */
#include "stm32f4xx.h"
#include "../FreeRTOS_Source/include/FreeRTOS.h"
#include "../FreeRTOS_Source/include/queue.h"
#include "../FreeRTOS_Source/include/semphr.h"
#include "../FreeRTOS_Source/include/task.h"
#include "../FreeRTOS_Source/include/timers.h"
#include "../FreeRTOS_Source/portable/MemMang/heap_4.c"






/*-----------------------------------------------------------*/
#define mainQUEUE_LENGTH 100


#define amber   0
#define green   1
#define red     2
#define blue    3


#define amber_led   LED3
#define green_led   LED4
#define red_led     LED5
#define blue_led    LED6






/* dd_task struct and task_type enum */
typedef enum task_type {PERIODIC,APERIODIC} task_type;


typedef struct dd_task {
   TaskHandle_t t_handle;
   task_type type;
   uint32_t task_id;
   uint32_t release_time;
   uint32_t absolute_deadline;
   uint32_t completion_time;
}dd_task;




/* Struct for linked list nodes */
typedef struct active_list {
   dd_task task;
   struct active_list *next_task;
}active_list;


/* queue_message struct for queues and message_type enum */
typedef enum message_type {RELEASE,COMPLETE, ACTIVE, COMPLETED, OVERDUE} message_type;


typedef struct queue_message {
   message_type msg_type;
   dd_task msg;
   uint32_t task_id;
}queue_message;


/*linked list heads */
active_list *head = NULL; //ACTIVE
active_list *head_c = NULL; //COMPLETE
active_list *head_o = NULL; //OVERDUE


//declarations
static void DD_Task_Generator1(void *pvParameters);
static void DD_Task_Generator2(void *pvParameters);
static void DD_Task_Generator3(void *pvParameters);
static void DD_Task_Suspend(void *pvParameters);
static void DD_Task_Scheduler(void *pvParameters);
static void User_Task_1(void *pvParameters);
static void User_Task_2(void *pvParameters);
static void User_Task_3(void *pvParameters);
static void DD_Task_Overdue(void *pvParameters);
static void DD_Monitor_Task(void *pvParameters);


void add_to_list(uint32_t list_type ,dd_task task_1);
void swap(active_list *a, active_list *b);
active_list* delete_first();
void complete_dd_task(TaskHandle_t t_handle, uint32_t task_id);
void update_overdue_list();


void ddCallback1( xTimerHandle xTimer );
void ddCallback2( xTimerHandle xTimer );
void ddCallback3( xTimerHandle xTimer );
void ddCallback4( xTimerHandle xTimer );
void ddCallbackMonitor( xTimerHandle xTimer );


xTimerHandle ddTimer1;
xTimerHandle ddTimer2;
xTimerHandle ddTimer3;
xTimerHandle ddTimer4;
xTimerHandle ddTimerMonitor;






xTimerHandle DD_Task_Gen_Timer;
xQueueHandle xQueue_main = 0;
xQueueHandle xQueue_actTasks = 0;
xQueueHandle xQueue_get = 0;


TaskHandle_t TaskHandle_1; //Task Generator 1
TaskHandle_t TaskHandle_2; //Task Generator 2
TaskHandle_t TaskHandle_3; //Task Generator 3
TaskHandle_t TaskHandle_DD; //DD Scheduler
TaskHandle_t TaskHandle_Monitor; //Monitor Task




/*-----------------------------------------------------------*/


// Timer callbacks which resume the corresponding task generator
void ddCallback1( xTimerHandle xTimer ) {
   vTaskResume(TaskHandle_1);
}


void ddCallback2( xTimerHandle xTimer ) {
   vTaskResume(TaskHandle_2);
}


void ddCallback3( xTimerHandle xTimer ) {
   vTaskResume(TaskHandle_3);
}


void ddCallbackMonitor( xTimerHandle xTimer ) {
   vTaskResume(TaskHandle_Monitor);
}








int main(void)
{


   /* Initialize LEDs */
   STM_EVAL_LEDInit(amber_led);
   STM_EVAL_LEDInit(green_led);
   STM_EVAL_LEDInit(red_led);
   STM_EVAL_LEDInit(blue_led);








   /* Create the queue used by the queue send and queue receive tasks.
   http://www.freertos.org/a00116.html */
   xQueue_main = xQueueCreate(     mainQUEUE_LENGTH,       /* The number of items the queue can hold. */
                           sizeof( queue_message * ) );    /* The size of each item the queue holds. */
   xQueue_actTasks = xQueueCreate(     mainQUEUE_LENGTH,       /* The number of items the queue can hold. */
                               sizeof( uint16_t ) );


   xQueue_get = xQueueCreate(  mainQUEUE_LENGTH,       /* The number of items the queue can hold. */
                           sizeof( active_list * ) );


   /* Add to the registry, for the benefit of kernel aware debugging. */
   vQueueAddToRegistry( xQueue_main, "MainQueue" );
   vQueueAddToRegistry( xQueue_get, "GetQueue" );


   xTaskCreate( DD_Task_Generator1, "DD_Generator1", configMINIMAL_STACK_SIZE, NULL, 2, &TaskHandle_1);
   xTaskCreate( DD_Task_Generator2, "DD_Generator2", configMINIMAL_STACK_SIZE, NULL, 2, &TaskHandle_2);
   xTaskCreate( DD_Task_Generator3, "DD_Generator3", configMINIMAL_STACK_SIZE, NULL, 2, &TaskHandle_3);
   xTaskCreate( DD_Task_Scheduler, "DD_Scheduler", configMINIMAL_STACK_SIZE, NULL, configMAX_PRIORITIES - 2, &TaskHandle_DD);


   xTaskCreate( DD_Monitor_Task, "DD_Monitor", configMINIMAL_STACK_SIZE, NULL, 1, &TaskHandle_Monitor);
   vTaskSuspend(TaskHandle_Monitor); //suspend monitor task


/*define the timers */
   ddTimer1 = xTimerCreate("ddTimer1", pdMS_TO_TICKS(500), pdTRUE, ( void * ) 0, ddCallback1);
   ddTimer2 = xTimerCreate("ddTimer2", pdMS_TO_TICKS(500), pdTRUE, ( void * ) 0, ddCallback2);
   ddTimer3 = xTimerCreate("ddTimer3", pdMS_TO_TICKS(500), pdTRUE, ( void * ) 0, ddCallback3);
   ddTimerMonitor = xTimerCreate("ddTimerMonitor", pdMS_TO_TICKS(1500), pdTRUE, ( void * ) 0, ddCallbackMonitor);




	/*Suspend task scheduler and start the task generator timers */
       vTaskSuspend(TaskHandle_DD);


       xTimerStart( ddTimer1, 0 );
       xTimerStart( ddTimer2, 0 );
       xTimerStart( ddTimer3, 0 );
       xTimerStart(ddTimerMonitor, 0);






   /* Start the tasks and timer running. */
   vTaskStartScheduler();


   return 0;
}




//helper function to add nodes to the linked list
//0 - active, 1 - completed, 2 - overdue
void add_to_list(uint32_t list_type ,dd_task task_1) {


   if(list_type == 0) {
       active_list* new_task = (active_list*) pvPortMalloc(sizeof(active_list));
       new_task->task = task_1;
       new_task->next_task = head;
       head = new_task;
   }
   else if(list_type == 1) {
       active_list* new_task = (active_list*) pvPortMalloc(sizeof(active_list));
       new_task->task = task_1;
       new_task->next_task = head_c;
       head_c = new_task;
   }
   else if (list_type == 2) {
       active_list* new_task = (active_list*) pvPortMalloc(sizeof(active_list));
       new_task->task = task_1;
       new_task->next_task = head_o;
       head_o = new_task;
   }
}




//helper function to print the linked list
void print_list(uint32_t list_type) {
   active_list* new_head;
   if(list_type == 0) {
       new_head = head;
   }
   else if(list_type == 1) {
           new_head = head_c;
       }
   else if(list_type == 2) {
       new_head = head_o;
   }


   while(new_head != NULL) {
       printf("%d ", new_head->task.task_id);
       new_head = new_head->next_task;
   }
}




/* Bubble sort to sort the linked list */
/* Code taken from : https://www.geeksforgeeks.org/c-program-bubble-sort-linked-list/ */
void sort_list() {
   active_list* new_head = head;
   int swapped;
   struct active_list* ptr1;
   struct active_list* lptr = NULL;


   /* Checking for empty list */
   if (new_head == NULL)
       return;


   do
   {
       swapped = 0;
       ptr1 = new_head;


       while (ptr1->next_task != lptr)
       {
           if (ptr1->task.absolute_deadline > ptr1->next_task->task.absolute_deadline)
           {
               swap(ptr1, ptr1->next_task);
               swapped = 1;
           }
           ptr1 = ptr1->next_task;
       }
       lptr = ptr1;
   }
   while (swapped);
}


//helper function used by the sort_list() function
//taken from: https://www.geeksforgeeks.org/c-program-bubble-sort-linked-list/
void swap(active_list *a, active_list *b)
{
   dd_task temp = a->task;
   a->task = b->task;
   b->task = temp;
}


/*helper function to delete linked list node */
active_list* delete_node(uint32_t t_id) {
   active_list *previous = NULL;
   active_list *current  = head;




   if(head == NULL){
       return NULL;
   }






   while(current->task.task_id != t_id) {
       if(current->next_task == NULL) {
           return NULL;
       }
       else {
           previous = current;
           current = current->next_task;
       }
    }


   if(current == head){
       head = head->next_task;
   }else{
       previous->next_task = current->next_task;
   }
   //print_list(0);
   return current;
}


//helper function which returns the first linked list node
active_list* return_first() {
   active_list *old_head = head;
   if(old_head == NULL) {
       return head;
   }
   else {
       return old_head;
   }
}


//helper function to return the linked list node with the specified ID
active_list* search_list(uint32_t t_id) {
   active_list* new_head = head;
   if(new_head == NULL) {
       return head;
   }
   else {
       while(new_head != NULL) {
           if(new_head->task.task_id == t_id) {
               return new_head;
           }
           new_head = new_head->next_task;
       }
   }
}


//Function to return the list length. Args ->  0 - active, 1 - complete, 2 - overdue
int listLength(uint32_t list_type){
   active_list* new_head;
   int size = 0;


       if(list_type == 0) {
           new_head = head;
       }
       if(list_type == 1) {
           new_head = head_c;
       }
       else if(list_type == 2) {
           new_head = head_o;
       }


       while(new_head != NULL) {
           ++size;
           new_head = new_head->next_task;
       }


   return(size);
}


//helper function to remove tasks from active list and add to the overdue list
void update_overdue_list() {
       active_list* new_head = head;
       while(new_head != NULL) {
           TickType_t curr_time = xTaskGetTickCount();
           //printf("%d vs. %d", curr_time, new_head->task.absolute_deadline);
           if(curr_time > new_head->task.absolute_deadline) {
               add_to_list(2, new_head->task);
               delete_node(new_head->task.task_id);
           }
           new_head = new_head->next_task;
       }
}


/*interface function called by the generator to create a dd_task struct and send message to the queue*/
void create_dd_task( TaskHandle_t t_handle, task_type type, uint32_t task_id, uint32_t absolute_deadline) {


   dd_task task1;


   task1.t_handle = t_handle;
   task1.type = type;
   task1.task_id = task_id;
   task1.absolute_deadline = absolute_deadline;
   task1.release_time = 0;
   task1.completion_time = 0;


   queue_message *send_msg = (queue_message *)pvPortMalloc(sizeof(queue_message));


   send_msg->msg_type = RELEASE;
   send_msg->msg = task1;


   xQueueSend(xQueue_main, &send_msg, 0);
   vTaskResume(TaskHandle_DD);
}


//interface function called by task when it finishes execution
void complete_dd_task(TaskHandle_t t_handle, uint32_t task_id) {
   //printf("execution completed!\n");
   queue_message *send_msg = (queue_message *)pvPortMalloc(sizeof(queue_message));


   send_msg->msg_type = COMPLETE;
   send_msg->task_id = task_id;
   //active_list* task_node = search_list(task_id);
   //vTaskDelete(t_handle);
   //vPortFree(&t_handle);
   xQueueSend(xQueue_main, &send_msg, 0);
   vTaskResume(TaskHandle_DD);
}


//Interface function which returns the number of items in the active list
uint32_t get_active_dd_task_list() {
   queue_message *send_msg = (queue_message *)pvPortMalloc(sizeof(queue_message));


   send_msg->msg_type = ACTIVE;
   xQueueSend(xQueue_main, &send_msg, 0);
   vTaskResume(TaskHandle_DD);


   active_list* active_head;


   xQueueSend(xQueue_main, &active_head, 0);
   uint32_t active_length = listLength(0);
   return active_length;
}


//Interface function which returns the number of items in the completed list
uint32_t get_complete_dd_task_list() {
   queue_message *send_msg = (queue_message *)pvPortMalloc(sizeof(queue_message));


   send_msg->msg_type = COMPLETE;
   xQueueSend(xQueue_main, &send_msg, 0);
   vTaskResume(TaskHandle_DD);


   active_list* complete_head;


   xQueueSend(xQueue_main, &complete_head, 0);
   uint32_t complete_length = listLength(1);
   return complete_length;
}


//Interface function which returns the number of items in the overdue list
uint32_t get_overdue_dd_task_list() {
   queue_message *send_msg = (queue_message *)pvPortMalloc(sizeof(queue_message));


   send_msg->msg_type = OVERDUE;
   xQueueSend(xQueue_main, &send_msg, 0);
   vTaskResume(TaskHandle_DD);


   active_list* overdue_head;


   xQueueSend(xQueue_main, &overdue_head, 0);
   uint32_t overdue_length = listLength(2);
   return overdue_length;
}


static void User_Task_1(void *pvParameters) {




   TickType_t current = xTaskGetTickCount();
   TickType_t previous;
   TickType_t exec = pdMS_TO_TICKS(100);


   STM_EVAL_LEDOn(amber_led);
   STM_EVAL_LEDOn(green_led);
   
   //get current tick on every iteration and decrement when not same
   while( exec > 0 )
   {
       previous = current;
       current = xTaskGetTickCount();


       if(current != previous)
       {
           exec--;
       }
   }


   STM_EVAL_LEDOff(amber_led);
   STM_EVAL_LEDOff(green_led);


   printf("Task 1 completed at time: %d \n", (int) xTaskGetTickCount());


   complete_dd_task(xTaskGetCurrentTaskHandle, 1);
   vTaskDelete(NULL);
}


static void User_Task_2(void *pvParameters) {




   TickType_t current = xTaskGetTickCount();
   TickType_t previous;
   TickType_t exec = pdMS_TO_TICKS(200);


   STM_EVAL_LEDOn(amber_led);
   STM_EVAL_LEDOn(green_led);


   //get current tick on every iteration and decrement when not same
   while( exec > 0 )
   {
       previous = current;
       current = xTaskGetTickCount();


       if(current != previous)
       {
           exec--;
       }
   }


   STM_EVAL_LEDOff(amber_led);
   STM_EVAL_LEDOff(green_led);


   printf("Task 2 completed at time: %d \n", (int) xTaskGetTickCount());


   complete_dd_task(xTaskGetCurrentTaskHandle, 2);
   vTaskDelete(NULL);
}


static void User_Task_3(void *pvParameters) {


   TickType_t current = xTaskGetTickCount();
   TickType_t previous;
   TickType_t exec = pdMS_TO_TICKS(200);


	\
   STM_EVAL_LEDOn(amber_led);
   STM_EVAL_LEDOn(green_led);


   //get current tick on every iteration and decrement when not same	
   while( exec > 0 )
   {
       previous = current;
       current = xTaskGetTickCount();


       if(current != previous)
       {
           exec--;
       }
   }


   STM_EVAL_LEDOff(amber_led);
   STM_EVAL_LEDOff(green_led);


   printf("Task 3 completed at time: %d \n", (int) xTaskGetTickCount());


   complete_dd_task(xTaskGetCurrentTaskHandle, 3);
   vTaskDelete(NULL);
}


//reports the current list items after every hyper period
static void DD_Monitor_Task(void *pvParameters) {


   TickType_t current;


   while(1) {
       current = xTaskGetTickCount();


       printf("\n");
       printf("\n");
       printf("$$$$$$$$$$ MONITOR TASK AT TIME: %d $$$$$$$$$$$$", (int) current);
       printf("\n");
       printf("\n");
       printf("####### Active DD-Tasks ######## \n");
       printf("\n Number of Tasks in Active DD Task List: %d \n", (int) get_active_dd_task_list());
       printf("\n Tasks currently in Active DD Task List: \n");
       print_list(0);
       printf("\n\n");
       printf("####### Completed DD-Tasks ##### \n");
       printf("\n Number of Tasks in Complete DD Task List: %d \n", (int) get_complete_dd_task_list());
       printf("\n Tasks currently in Complete DD Task List: \n");
       print_list(1);
       printf("\n\n");
       printf("####### Overdue DD-Tasks ####### \n");
       printf("\n Number of Tasks in Overdue DD Task List: %d \n", (int) get_overdue_dd_task_list());
       printf("\n Tasks currently in Overdue DD Task List: \n");
       print_list(2);
       printf("\n\n");


       vTaskSuspend(NULL); //suspend itself when done


   }


}


//init function —-unused
static void DD_Task_Suspend(void *pvParameters) {
   vTaskSuspend(TaskHandle_1);
   vTaskSuspend(TaskHandle_2);
   vTaskSuspend(TaskHandle_3);
   vTaskSuspend(TaskHandle_DD);


   xTimerStart( ddTimer1, 0 );
   xTimerStart( ddTimer2, 0 );
   xTimerStart( ddTimer3, 0 );
   xTimerStart( ddTimer4, 0 );
   vTaskDelete(NULL);
}


static void DD_Task_Generator1(void *pvParameters) {
   while(1) {
       TaskHandle_t TaskHandle_User1;


       BaseType_t xReturned = xTaskCreate( User_Task_1, "User_1", configMINIMAL_STACK_SIZE, NULL, tskIDLE_PRIORITY, &TaskHandle_User1);




       if(xReturned != pdPASS) {
           printf("task init failed!");
       }


       create_dd_task(TaskHandle_User1, PERIODIC, 1, xTaskGetTickCount() + pdMS_TO_TICKS(500));


       vTaskSuspend(NULL); //suspend itself
   }
}


static void DD_Task_Generator2(void *pvParameters) {
   while(1) {
           TaskHandle_t TaskHandle_User2;


           BaseType_t xReturned = xTaskCreate( User_Task_2, "User_2", configMINIMAL_STACK_SIZE, NULL, tskIDLE_PRIORITY, &TaskHandle_User2);




           if(xReturned != pdPASS) {
               printf("task init failed!");
           }


           create_dd_task(TaskHandle_User2, PERIODIC, 2, xTaskGetTickCount() + pdMS_TO_TICKS(500));


           vTaskSuspend(NULL); //suspend itself
       }
}


static void DD_Task_Generator3(void *pvParameters) {
   while(1) {
           TaskHandle_t TaskHandle_User3;


           BaseType_t xReturned = xTaskCreate( User_Task_3, "User_3", configMINIMAL_STACK_SIZE, NULL, tskIDLE_PRIORITY, &TaskHandle_User3);


           if(xReturned != pdPASS) {
               printf("task init failed!");
           }


           create_dd_task(TaskHandle_User3, PERIODIC, 3, xTaskGetTickCount() + pdMS_TO_TICKS(500));


           vTaskSuspend(NULL); //suspend itself
       }
}


//Task Scheduler —- highest priority when not suspended
static void DD_Task_Scheduler(void *pvParameters) {
   while(1) {
       queue_message* rcvd_msg = (queue_message *)pvPortMalloc(sizeof(queue_message));
       if(!xQueueReceive(xQueue_main, &rcvd_msg, 500)) {
           continue;
       }


       update_overdue_list(); //remove overdue tasks


       message_type rcvd_type = rcvd_msg->msg_type;


       if(rcvd_type == RELEASE) {
           dd_task rcvd_task = rcvd_msg->msg;


           if(rcvd_task.task_id < 4) {


           printf("Task %d released at time: %d \n",  rcvd_task.task_id, (int) xTaskGetTickCount());


           rcvd_task.release_time = xTaskGetTickCount();
           add_to_list(0, rcvd_task);


           sort_list();


           active_list* erlst_task = return_first(); //get earliest task


           vTaskPrioritySet(erlst_task->task.t_handle , configMAX_PRIORITIES - 3); 
           vPortFree(rcvd_msg);
           vTaskSuspend(NULL);
           }
       }
       else if(rcvd_type == COMPLETE) {


           uint32_t rcvd_id = rcvd_msg->task_id;
           if(rcvd_id < 4) {


           active_list* task_node = search_list(rcvd_id);
           uint32_t cmplt_time = xTaskGetTickCount();


           task_node->task.completion_time = cmplt_time;
           uint32_t deadline = task_node->task.absolute_deadline;


           active_list* deleted_node =  delete_node(rcvd_id);
           if(deleted_node == NULL) {
               //printf("null returned\n");
           }
           if(deleted_node != NULL) {


               add_to_list(1, task_node->task);
               sort_list();
               vPortFree(rcvd_msg);
               vTaskSuspend(NULL);
           }
           }
       }
       else if(rcvd_type == ACTIVE) {
           active_list *act = head;
           xQueueSend(xQueue_main, &act, 0);
       }
       else if(rcvd_type == COMPLETE) {
           active_list *cmplt = head_c;
           xQueueSend(xQueue_main, &cmplt, 0);
       }
       else if(rcvd_type == OVERDUE) {
           active_list *overd = head_o;
           xQueueSend(xQueue_main, &overd, 0);
       }




   }
}


/*-----------------------------------------------------------*/


/*-----------------------------------------------------------*/


void vApplicationMallocFailedHook( void )
{
   /* The malloc failed hook is enabled by setting
   configUSE_MALLOC_FAILED_HOOK to 1 in FreeRTOSConfig.h.


   Called if a call to pvPortMalloc() fails because there is insufficient
   free memory available in the FreeRTOS heap.  pvPortMalloc() is called
   internally by FreeRTOS API functions that create tasks, queues, software
   timers, and semaphores.  The size of the FreeRTOS heap is set by the
   configTOTAL_HEAP_SIZE configuration constant in FreeRTOSConfig.h. */
   for( ;; );
}
/*-----------------------------------------------------------*/


void vApplicationStackOverflowHook( xTaskHandle pxTask, signed char *pcTaskName )
{
   ( void ) pcTaskName;
   ( void ) pxTask;


   /* Run time stack overflow checking is performed if
   configconfigCHECK_FOR_STACK_OVERFLOW is defined to 1 or 2.  This hook
   function is called if a stack overflow is detected.  pxCurrentTCB can be
   inspected in the debugger if the task name passed into this function is
   corrupt. */
   for( ;; );
}
/*-----------------------------------------------------------*/


void vApplicationIdleHook( void )
{
volatile size_t xFreeStackSpace;


   /* The idle task hook is enabled by setting configUSE_IDLE_HOOK to 1 in
   FreeRTOSConfig.h.


   This function is called on each cycle of the idle task.  In this case it
   does nothing useful, other than report the amount of FreeRTOS heap that
   remains unallocated. */
   xFreeStackSpace = xPortGetFreeHeapSize();


   if( xFreeStackSpace > 100 )
   {
       /* By now, the kernel has allocated everything it is going to, so
       if there is a lot of heap remaining unallocated then
       the value of configTOTAL_HEAP_SIZE in FreeRTOSConfig.h can be
       reduced accordingly. */
   }
}
