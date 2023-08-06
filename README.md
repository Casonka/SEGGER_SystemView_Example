# Рабочий пример файлов для мониторинга работы ОСРВ FreeRTOS через программу SystemView Free
Материалы и инструкция по установке были взяты [отсюда](https://www.youtube.com/watch?v=QEAvmTgdWMs&t=374s) (eng. Thank you for installation guide Vidura)
Проект создан для сохранения быстро доступной инструкции по настройке проекта. Программа предоставляет инструментарий для повышения отладочной способности ОСРВ и межпотокового взаимодействия

Необходимое оборудование:
- STM32CubeIDE, Eclipse IDE (опционально)
- Целевое устройство (микроконтроллер) с прошивкой ОСРВ (FreeRTOS, EmbOS и др.)
- Программатор [J-Link (V9)](https://www.segger.com/products/debug-probes/j-link/#models) либо программатор ST-LinkV2 [переделанный](https://cdeblog.ru/converting-st-link-into-a-j-link) в J-Link
- Установленная программа [SystemView](https://www.segger.com/products/development-tools/systemview/)

# Инструкция по установке

1) Зайти на страницу загрузки необходимого ПО - [ссылка](https://www.youtube.com/watch?v=QEAvmTgdWMs&t=374s)
2) После установки SystemView копируем репозиторий через ``` git clone https://github.com/Casonka/SEGGER_SystemView_Examle.git ```
3) Переименовать папку SEGGER_SystemView_Example в SEGGER (опционально)
4) Переместить папку в ваш проект (для CubeIDE это может быть папка Middlewares)
5) Добавить пути в includes and paths (для CubeIDE и Eclipse зайти в меню можно через Project->Properties->C/C++ General/Paths and Symbols)
6) В окне includes вставьте следующие строки (заменим PROJ на папку с вашим проектом):
   ```
   /PROJ/Middlewares/SEGGER/SEGGER
   /PROJ/Middlewares/SEGGER/OS
   /PROJ/Middlewares/SEGGER/Config
   ```
7) Для применения специального патча для FreeRTOS найти соответствующую папку в Middlewares и изменить содержимое ```Middlewares/Third_Party/FreeRTOS/Source``` на ```Middlewares/Third_Party/FreeRTOS/org/Source```
8) Применить патч через ПКМ по проекту -> Team -> Apply Patch, выберите файл, находящийся в папке Patch и нажимайте Next, выбрав директорию FreeRTOS. Следуйте далее до менеджера слияния изменений, нажимайте Finish
9) Для работы стриминга и передачи данных в SystemView сделайте ```#include "SEGGER_SYSVIEW.h"``` в main.c или freertos.c и используйте команды ```SEGGER_SYSVIEW_Conf(); SEGGER_SYSVIEW_Start();``` для запуска процесса мониторинга (выполнить команды желательно максимально рано, насколько возможно и до запуска и резервирования объектов отслеживаемой ОСРВ)
10) Зайти в файл SEGGER_SYSVIEW_Config_FreeRTOS.c и изменить следующие параметры:
    ```
    SYSVIEW_APP_NAME - отображаемое название программы в SystemView
    SYSVIEW_DEVICE_NAME - используемая архитектура микроконтроллера (например Cortex-M3)
    SYSVIEW_RAM_BASE - начальный адрес оперативной памяти используемого контроллера (отображено в программе STM32CubeIDE, можно найти в даташите) 
    ```
12) ВАЖНО! В некоторых случаях выполнение этих двух команд приводят к ошибке 	```configASSERT( ( portAIRCR_REG & portPRIORITY_GROUP_MASK ) <= ulMaxPRIGROUPValue );``` которая связана с группировкой приоритетов. Чтобы решить эту проблему нужно следовать следующей [инструкции](https://forum.segger.com/index.php/Thread/6046-SOLVED-Systemview-stuck-in-configASSERT-with-FreeRTOS-STM32CubeMX/). Вот код который необходимо добавить, чтобы исключить эту ошибку:
```
//###########################################################
// Add function below in file portmacro.h
#ifdef configASSERT
	void vSetVarulMaxPRIGROUPValue( void );
#endif
//###########################################################
// Add function below in file port.c
#if( configASSERT_DEFINED == 1 )
void vSetVarulMaxPRIGROUPValue( void )
{
	volatile uint8_t * const pucFirstUserPriorityRegister = ( volatile uint8_t * const ) ( portNVIC_IP_REGISTERS_OFFSET_16 + portFIRST_USER_INTERRUPT_NUMBER );
	volatile uint8_t ucMaxPriorityValue;
	/* Determine the number of priority bits available.  First write to all
	possible bits. */
	*pucFirstUserPriorityRegister = portMAX_8_BIT_VALUE;
	/* Read the value back to see how many bits stuck. */
	ucMaxPriorityValue = *pucFirstUserPriorityRegister;
	/* Calculate the maximum acceptable priority group value for the number
	of bits read back. */
	ulMaxPRIGROUPValue = portMAX_PRIGROUP_BITS;
	while( ( ucMaxPriorityValue & portTOP_BIT_OF_BYTE ) == portTOP_BIT_OF_BYTE )
	{
		ulMaxPRIGROUPValue--;
		ucMaxPriorityValue <<= ( uint8_t ) 0x01;
	}
#ifdef __NVIC_PRIO_BITS
	{
		/* Check the CMSIS configuration that defines the number of
		priority bits matches the number of priority bits actually queried
		from the hardware. */
		configASSERT( ( portMAX_PRIGROUP_BITS - ulMaxPRIGROUPValue ) == __NVIC_PRIO_BITS );
	}
#endif
#ifdef configPRIO_BITS
	{
		/* Check the FreeRTOS configuration that defines the number of
		priority bits matches the number of priority bits actually queried
		from the hardware. */
		configASSERT( ( portMAX_PRIGROUP_BITS - ulMaxPRIGROUPValue ) == configPRIO_BITS );
	}
#endif
	/* Shift the priority group value back to its position within the AIRCR
	register. */
	ulMaxPRIGROUPValue <<= portPRIGROUP_SHIFT;
	ulMaxPRIGROUPValue &= portPRIORITY_GROUP_MASK;
}
#endif /* conifgASSERT_DEFINED */
```
13) После добавления новой функции добавить её согласно следующему приведенному фрагменту кода:
    ```
	SEGGER_SYSVIEW_Conf();
	vSetVarulMaxPRIGROUPValue();
	SEGGER_SYSVIEW_Start();
    ```
14) Запускаем SystemView, подключаем J-Link к контроллеру, запускаем отладку и начинаем запись, данные будут немедленно поступать в интерфейс программы.
