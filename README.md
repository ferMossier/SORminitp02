# SOR-Semáforos-1S-2021 - Mossier Fernando

*Trabajo Práctico Semáforos primer semestre 2022*

---

### Docentes

- Mariano Vargas
- Ignacio Tula

---

## Link

Desde el siguiente link será posible descargar el código fuente y toda la documentación sobre el trabajo práctico:

[LINK: MiniTp02-MossierFernando](https://github.com/ferMossier/SORminitp02)

### Compilar
- gcc subwayArgento.c -o subway -lpthread
### Ejecutar
- ./subway

---

## Introduccion

El presente trabajo se propone simular una competencia culinaria entre cuatro equipos. El trabajo de cada uno de los equipos consiste en seguir una receta paso a paso para realizar un sandwich de milanesa completo. 

Los equipos deberán turnarse para el uso de la sartén, el horno y la sal ya que solo existe uno de cada uno para los cuatro equipos.

El objetivo es armar el sandwich correctamente en el menor tiempo posible. El primer equipo en lograrlo será el ganador

---

## Aspectos técnicos

Con respecto a los aspectos técnicos de realización cabe destacar que:
- Se utilizará el lenguaje de programación C.
- Para simular el trabajo en paralelo entre los cuatro equipos y para simular el trabajo en paralelo de las tareas internas de cada equipo se hará uso de Threads (hilos)
- Para las tareas internas de cada equipo cuya realización dependan de la finalización de otras tareas y para las tareas que requieran el uso de recursos compartidos (sartén, salero y horno) se utilizarán semáforos.
- Al finalizar la ejecución del programa se podrá consultar el archivo *log.txt* para ver un informe detallado del proceso de cada equipo y conocer al ganador.

---
## Receta

1. Cortar dos dientes de ajo y perejil.
2. Mezclar los dos dientes de ajo y el perejil picado con huevo y carne.
3. Salar la mezcla anterior a gusto.
4. Empanar las milanesas.
5. Cocinarlas por 5 minutos en sarten.
6. Hornear el pan por diez minutos.
7. Cortar lechuga, tomate, ceboola morada y pepino.
8. Armar sandwich.

---

## Codigo

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

char *archivo_nombre = "log.txt";
char *modo = "a+";

sem_t sem_horno;
sem_t sem_salero;
sem_t sem_sarten;
sem_t sem_ganar;

char *terminar_estado = "terminar";
int ganar_estado = 1;

#define LIMITE 50

struct semaforos
{
	sem_t sem_mezclar;
	sem_t sem_salar;
	sem_t sem_empanar;
	sem_t sem_cocinar;
	sem_t sem_cortar_verduras;
	sem_t sem_pan;
	sem_t sem_armar;
	sem_t sem_terminar;
};

struct paso
{
	char accion[LIMITE];
	char ingredientes[4][LIMITE];
};

struct parametro
{
	int equipo_param;
	struct semaforos semaforos_param;
	struct paso pasos_param[8];
};

void *imprimirAccion(void *data, char *accionIn)
{
	struct parametro *mydata = data;
	FILE *archivo = fopen(archivo_nombre, modo);
	if (strcmp(terminar_estado, accionIn) == 0)
	{
		printf("\tEquipo %d - termino! \n ", mydata->equipo_param);
		fprintf(archivo, "\tEquipo %d - termino! \n ", mydata->equipo_param);
		sem_wait(&sem_ganar);
		if (ganar_estado == 1)
		{
			ganar_estado = 0;
			printf("\tEquipo %d - ***¡GANADOR!***! \n ", mydata->equipo_param);
			fprintf(archivo, "\tEquipo %d - ***¡GANADOR!*** \n ", mydata->equipo_param);
		}
	}
	int sizeArray = (int)(sizeof(mydata->pasos_param) / sizeof(mydata->pasos_param[0]));
	int i;
	for (i = 0; i < sizeArray; i++)
	{
		if (strcmp(mydata->pasos_param[i].accion, accionIn) == 0)
		{
			printf("\tEquipo %d - accion %s \n ", mydata->equipo_param, mydata->pasos_param[i].accion);
			fprintf(archivo, "\tEquipo %d - accion %s \n ", mydata->equipo_param, mydata->pasos_param[i].accion);
			int sizeArrayIngredientes = (int)(sizeof(mydata->pasos_param[i].ingredientes) / sizeof(mydata->pasos_param[i].ingredientes[0]));
			int h;
			printf("\tEquipo %d ----------- ingredientes : ----------\n", mydata->equipo_param);
			fprintf(archivo, "\tEquipo %d ----------- ingredientes : ----------\n", mydata->equipo_param);
			for (h = 0; h < sizeArrayIngredientes; h++)
			{
				if (strlen(mydata->pasos_param[i].ingredientes[h]) != 0)
				{
					printf("\tEquipo %d ingrediente  %d : %s \n", mydata->equipo_param, h, mydata->pasos_param[i].ingredientes[h]);
					fprintf(archivo, "\tEquipo %d ingrediente  %d : %s \n", mydata->equipo_param, h, mydata->pasos_param[i].ingredientes[h]);
				}
			}
		}
	}
}

void *cortar(void *data)
{
	char *accion = "cortar";
	struct parametro *mydata = data;
	imprimirAccion(mydata, accion);
	usleep(1000000);
	sem_post(&mydata->semaforos_param.sem_mezclar);
	pthread_exit(NULL);
}

void *cortarVerduras(void *data)
{
	char *accion = "cortar otros";
	struct parametro *mydata = data;
	imprimirAccion(mydata, accion);
	usleep(1000000);
	sem_post(&mydata->semaforos_param.sem_cortar_verduras);
	pthread_exit(NULL);
}

void *mezclar(void *data)
{
	char *accion = "mezclar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_mezclar);
	imprimirAccion(mydata, accion);
	usleep(2000000);
	sem_post(&mydata->semaforos_param.sem_salar);
	pthread_exit(NULL);
}

void *salar(void *data)
{
	char *accion = "salar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_salar);
	sem_wait(&sem_salero);
	imprimirAccion(mydata, accion);
	usleep(500000);
	sem_post(&mydata->semaforos_param.sem_empanar);
	sem_post(&sem_salero);
	pthread_exit(NULL);
}

void *empanar(void *data)
{
	char *accion = "empanar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_empanar);
	imprimirAccion(mydata, accion);
	usleep(2000000);
	sem_post(&mydata->semaforos_param.sem_cocinar);
	pthread_exit(NULL);
}

void *cocinar(void *data)
{
	char *accion = "cocinar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_cocinar);
	sem_wait(&sem_sarten);
	imprimirAccion(mydata, accion);
	usleep(5000000);
	sem_post(&mydata->semaforos_param.sem_pan);
	sem_post(&sem_sarten);
	pthread_exit(NULL);
}

void *hornear(void *data)
{
	char *accion = "hornear";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_pan);
	sem_wait(&sem_horno);
	imprimirAccion(mydata, accion);
	usleep(8000000);
	sem_post(&sem_horno);
	sem_post(&mydata->semaforos_param.sem_armar);
	pthread_exit(NULL);
}

void *armar(void *data)
{
	char *accion = "armar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_cortar_verduras);
	sem_wait(&mydata->semaforos_param.sem_armar);
	imprimirAccion(mydata, accion);
	usleep(3000000);
	sem_post(&mydata->semaforos_param.sem_terminar);
	pthread_exit(NULL);
}

void *terminar(void *data)
{
	char *accion = "terminar";
	struct parametro *mydata = data;
	sem_wait(&mydata->semaforos_param.sem_terminar);
	sem_post(&sem_ganar);
	imprimirAccion(mydata, accion);
	pthread_exit(NULL);
}

void *ejecutarReceta(void *i)
{
	sem_t sem_mezclar;
	sem_t sem_salar;
	sem_t sem_empanar;
	sem_t sem_cocinar;
	sem_t sem_pan;
	sem_t sem_cortar_verduras;
	sem_t sem_armar;
	sem_t sem_terminar;

	pthread_t p1;
	pthread_t p2;
	pthread_t p3;
	pthread_t p4;
	pthread_t p5;
	pthread_t p6;
	pthread_t p7;
	pthread_t p8;
	pthread_t p9;

	int p = *((int *)i);

	printf("Ejecutando equipo %d \n", p);

	struct parametro *pthread_data = malloc(sizeof(struct parametro));

	pthread_data->equipo_param = p;

	pthread_data->semaforos_param.sem_mezclar = sem_mezclar;
	pthread_data->semaforos_param.sem_salar = sem_salar;
	pthread_data->semaforos_param.sem_empanar = sem_empanar;
	pthread_data->semaforos_param.sem_cocinar = sem_cocinar;
	pthread_data->semaforos_param.sem_pan = sem_pan;
	pthread_data->semaforos_param.sem_cortar_verduras = sem_cortar_verduras;
	pthread_data->semaforos_param.sem_armar = sem_armar;
	pthread_data->semaforos_param.sem_terminar = sem_terminar;

	strcpy(pthread_data->pasos_param[0].accion, "cortar");
	strcpy(pthread_data->pasos_param[0].ingredientes[0], "ajo");
	strcpy(pthread_data->pasos_param[0].ingredientes[1], "perejil");
	strcpy(pthread_data->pasos_param[1].accion, "mezclar");
	strcpy(pthread_data->pasos_param[1].ingredientes[0], "ajo");
	strcpy(pthread_data->pasos_param[1].ingredientes[1], "perejil");
	strcpy(pthread_data->pasos_param[1].ingredientes[2], "huevo");
	strcpy(pthread_data->pasos_param[1].ingredientes[3], "carne");
	strcpy(pthread_data->pasos_param[2].accion, "salar");
	strcpy(pthread_data->pasos_param[2].ingredientes[0], "sal");
	strcpy(pthread_data->pasos_param[2].ingredientes[1], "mezcla");
	strcpy(pthread_data->pasos_param[3].accion, "empanar");
	strcpy(pthread_data->pasos_param[3].ingredientes[0], "pan rallado");
	strcpy(pthread_data->pasos_param[3].ingredientes[1], "carne");
	strcpy(pthread_data->pasos_param[3].ingredientes[2], "mezcla");
	strcpy(pthread_data->pasos_param[4].accion, "cocinar");
	strcpy(pthread_data->pasos_param[4].ingredientes[0], "milanesa");
	strcpy(pthread_data->pasos_param[5].accion, "hornear");
	strcpy(pthread_data->pasos_param[5].ingredientes[0], "panes");
	strcpy(pthread_data->pasos_param[6].accion, "cortar otros");
	strcpy(pthread_data->pasos_param[6].ingredientes[0], "lechuga");
	strcpy(pthread_data->pasos_param[6].ingredientes[1], "tomate");
	strcpy(pthread_data->pasos_param[6].ingredientes[2], "cebolla morada");
	strcpy(pthread_data->pasos_param[6].ingredientes[3], "pepino");
	strcpy(pthread_data->pasos_param[7].accion, "armar");
	strcpy(pthread_data->pasos_param[7].ingredientes[0], "milanesa");
	strcpy(pthread_data->pasos_param[7].ingredientes[1], "pan");
	strcpy(pthread_data->pasos_param[7].ingredientes[2], "verduras");

	sem_init(&(pthread_data->semaforos_param.sem_mezclar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_salar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_empanar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_cocinar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_cortar_verduras), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_pan), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_armar), 0, 0);
	sem_init(&(pthread_data->semaforos_param.sem_terminar), 0, 0);

	int rc;
	rc = pthread_create(&p1, NULL, cortar, pthread_data);
	rc = pthread_create(&p2, NULL, mezclar, pthread_data);
	rc = pthread_create(&p3, NULL, salar, pthread_data);
	rc = pthread_create(&p4, NULL, empanar, pthread_data);
	rc = pthread_create(&p5, NULL, cocinar, pthread_data);
	rc = pthread_create(&p7, NULL, cortarVerduras, pthread_data);
	rc = pthread_create(&p6, NULL, hornear, pthread_data);
	rc = pthread_create(&p8, NULL, armar, pthread_data);
	rc = pthread_create(&p9, NULL, terminar, pthread_data);

	pthread_join(p1, NULL);
	pthread_join(p2, NULL);
	pthread_join(p3, NULL);
	pthread_join(p4, NULL);
	pthread_join(p5, NULL);
	pthread_join(p6, NULL);
	pthread_join(p7, NULL);
	pthread_join(p8, NULL);
	pthread_join(p9, NULL);

	if (rc)
	{
		printf("Error:unable to create thread, %d \n", rc);
		exit(-1);
	}

	sem_destroy(&sem_mezclar);
	sem_destroy(&sem_salar);
	sem_destroy(&sem_empanar);
	sem_destroy(&sem_cocinar);
	sem_destroy(&sem_cortar_verduras);
	sem_destroy(&sem_pan);
	sem_destroy(&sem_armar);
	sem_destroy(&sem_terminar);

	pthread_exit(NULL);
}

int main()
{
	int rc;
	int *equipoNombre1 = malloc(sizeof(*equipoNombre1));
	int *equipoNombre2 = malloc(sizeof(*equipoNombre2));
	int *equipoNombre3 = malloc(sizeof(*equipoNombre3));
	int *equipoNombre4 = malloc(sizeof(*equipoNombre4));

	*equipoNombre1 = 1;
	*equipoNombre2 = 2;
	*equipoNombre3 = 3;
	*equipoNombre4 = 4;

	pthread_t equipo1;
	pthread_t equipo2;
	pthread_t equipo3;
	pthread_t equipo4;

	sem_init((&sem_horno), 0, 1);
	sem_init((&sem_salero), 0, 1);
	sem_init((&sem_sarten), 0, 1);
	sem_init((&sem_ganar), 0, 0); 

	rc = pthread_create(&equipo1, NULL, ejecutarReceta, equipoNombre1);
	rc = pthread_create(&equipo2, NULL, ejecutarReceta, equipoNombre2);
	rc = pthread_create(&equipo3, NULL, ejecutarReceta, equipoNombre3);
	rc = pthread_create(&equipo4, NULL, ejecutarReceta, equipoNombre4);

	if (rc)
	{
		printf("Error:unable to create thread, %d \n", rc);
		exit(-1);
	}

	pthread_join(equipo1, NULL);
	pthread_join(equipo2, NULL);
	pthread_join(equipo3, NULL);
	pthread_join(equipo4, NULL);

	sem_destroy(&sem_horno);
	sem_destroy(&sem_salero);
	sem_destroy(&sem_sarten);
	sem_destroy(&sem_ganar);

	pthread_exit(NULL);
}
```

---

## Ejecución del programa

Lo siguiente es una breve explicación sobre como se comporta el programa

1. El programa inicia ejecutando cuatro hilos, uno por cada equipo. Es decir que estos equipos estarán haciendo concurrentemente todas las tareas que sean posibles.
2. Al interior de cada equipo, cada tarea se ejecutará en un hilo independiente y su ejecución estará organizada con semáforos ya que hay tareas que dependen de la finalización de otras. 
3. Casi todas las tareas son dependientes de otras a excepción de dos:
   - cortar ajo y perejil --> Se ejecuta apenas inicia la ejecución
   - cortar lechuga, tomate, cebolla y pepino --> Se ejecuta en cualquier momento disponible de forma concurrente con otras tareas.
4. Los equipos van a disputarse el uso de los recursos compartidos (sarten, sal y horno), por lo tanto el ganador será aleatorio ya que dependerá del scheduler. Multiples ejecuciones del programa arrojarán diferentes ganadores.
5. Finalmente, el primer equipo que logre finalizar todas las tareas primero, será el ganador.

**Para mas detalle, ver diagrama de ejecución adjunto.**

---

## Problemas y sus soluciones

Al momento de desarrollar lo solicitado en la consigna, el primer paso fue investigar y consultar documentación sobre la librería *semaphore.h* de C, ya que desconocía las funciones que se estaban utilizando y que significaban sus parámetros. 

Una vez investigado, probado y entendido como funciona esa librería, decidí ponerme a codificar tomando como referencia el template que nos fue otorgado. Desde ya que resultó muy complicado ya que había hecho un análisis muy escaso de los requerimientos y de como pretendía que funcione el programa. Es por esto que decidí bosquejar un diagrama sobre como debería comportarse la ejecución. Eso me ayudó mucho a aclarar como organizar el código.

Por otro lado, el trabajo me presentó algunos problemas al momento de ejecutar hilos y semáforos ya que el mismo se comportaba erráticamente y varias veces la ejecución se quedaba colgada. Llegué a la conclusión de que eso tenía que ver con que había funciones que se quedaban esperando la liberación de semáforos que nunca se liberaban. Después de mucho análisis y correcciones se llegó a una versión que funciona correctamente.

---

## Conclusiones

La realización de este trabajo tuvo en mi dos grandes beneficios: en primer lugar me resulta muy agradable empezar a conocer un lenguaje de programación de más bajo nivel con respecto a los que estoy acostumbrado y, en segundo lugar, me ayudó muchísimo a entender mejor los beneficios de la programación concurrente. En un programa que se ejecuta en múltiples hilos, los tiempos de ejecución se reducen muchísimo cuando es necesario ejecutar varias tareas independientes entre sí. Y en el caso de que existan tareas que son dependientes entre esos hilos, fácilmente se pueden organizar mediante semáforos. 
