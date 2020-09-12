El objetivo de este pagina es mostrar el paso a paso para realizar la integración de Red Hat Insights con Ansible Tower y asi garantizar que los sistemas operativos de linea base cuenten por lo menos con un aseguramiento mínimo sugerido por algunas de las normas internacionales.

Para ello es necesario realizar los siguientes pasos:

## Registrar las servidores a Red Hat Insights

1.  Suscribir maquina con su usuario y contraseña de Red Hat a traves del comando subscription-manager
```bash
[root@node01 ~]# subscription-manager register
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: jcalvo@redhat.com
Password:
The system has been registered with ID: 8c00b1e6-dc02-4363-badd-3b10858f3379
The registered system name is: node01.jmanuelcalvo.com
```

2. Adjuntar el sistema operativo a una suscripción activa
```bash
[root@node01 ~]# subscription-manager attach --pool=XXXXXXXXXXXXXXXXXXX
Successfully attached a subscription for: Employee SKU
```
> Nota: 
> 
> En caso de no conocer el pool de la suscripción puede ejecutar el comando `subscription-manager list --available` 

3. Habilitar el repositorio donde se encuentra los paquetes de Red Hat Insights
```bash
[root@node01 ~]# subscription-manager repos --enable=rhel-7-server-insights-3-rpms
Repository 'rhel-7-server-insights-3-rpms' is enabled for this system.
```
> Nota:
>
> Este paso es unicamente requerido para *RHEL 7*, ya que *RHEL 8* ya tiene habilitados los paquetes de insights por defecto
>
> En caso que quiera validar los múltiples repositorios dados por la suscripción puede ejecutar el comando `subscription-manager repos --list`

4. Instalar el paquete de Red Hat Insights
```bash
[root@node01 ~]# yum install insights-client
...
...
Installed:
  insights-client.noarch 0:3.0.13-1.el7_7

Complete!
```

5. Registrar la maquina a Red Hat Insights
```bash
[root@node01 ~]# insights-client --register
Successfully registered host node01.jmanuelcalvo.com
Automatic scheduling for Insights has been enabled.
Starting to collect Insights data for node01.jmanuelcalvo.com
Uploading Insights data.
Successfully uploaded report for node01.jmanuelcalvo.com.
View the Red Hat Insights console at https://cloud.redhat.com/insights/
View details about this system on cloud.redhat.com:
https://cloud.redhat.com/insights/inventory/1616ef27-d477-40ae-9c4e-a9c4f29b78f0
```

6. Ingresar al portal de cloud.redhat.com y validar dentro del inventario que las maquinas quedaron registradas, puede ingresar al portal principal del Red Hat Insights o al url devuelto por el comando de registro, el cual lo llevara directamente al inventario

https://cloud.redhat.com/insights/
https://cloud.redhat.com/insights/inventory/1616ef27-d477-40ae-9c4e-xxxxx

![Ref](img/insights1.png)

> Consejo
> 
> Si no quiere realizar estos pasos de forma manual o requiere realizarlos sobre múltiples servidores, puede hacerlo a través de este playbook

```yaml
---
- hosts: all
  gather_facts: yes
  tasks:
  - name: Registar las maquinas al pool XXXXX SKU
    redhat_subscription:
      state: present
      username: jcalvo@redhat.com
      password: XXXXXXX
      pool_ids: XXXXXXXXXXXXXXXXXXX

  - name: Habiliar repositorio de insights
    rhsm_repository:
      name: rhel-7-server-insights-3-rpms

  - name: Instalar paquete de Red Hat Insights
    yum:
      name: insights-client
      state: latest

  - name: Registrar maquina en insights
    shell: insights-client --register
    register: salida

  - name: Mensaje de salida registro
    debug:
      var: salida  
```

## Registrar los nodos en Ansible Tower


1. Registrar el usuario y contraseña utilizados en Red Hat Insights dentro del Ansible Tower, para ello desde la interface de Ansible Tower ir a:

Credentials -> Adicionar credenciales  ![plus](img/green_plus.png)

En el campo **Tipo de credencial**, ingrese Insights o haga clic en el botón y selecciónelo en la ventana emergente del tipo de credencial.

![Ref](img/tower-creds.png)

y rellene los campos con los mismos datos de Registro de la maquinas y los del portal **Red Hat Customer Portal**

Haga clic en **Save** cuando termine


2. Crear un proyecto de integración con Red Hat Insights, desde la interface de Ansible Tower ir a:

Projects -> Adicionar proyecto ![plus](img/green_plus.png)

El campo **Credential** se rellena previamente con las credenciales de Insights que creó anteriormente. De lo contrario, ingrese la credencial o haga clic en el botón y selecciónelo en la ventana emergente.

![Ref](img/tower-project.png)

Haga clic en **Save** cuando termine

Una vez creado el proyecto, este intentara sincronizar con los playbooks que se tengan creados en el portal cloud.redhat.com

> NOTA:
>
> Puede suceder que la primera sincronización falle por timeout, intente nuevamente sincronizar de forma manual luego de 5 minutos

![Ref](img/tower-project1.png)

3. Crear un inventario con los hosts ingresados a Red Hat Insights para la integración, desde la interface de Ansible Tower ir a:

Inventories -> ![plus](img/green_plus.png) -> Inventory


![Ref](img/tower-inventory.png)

Haga clic en **Save** cuando termine

Una vez creado el inventario, se deben vincular los hosts previamente registrados, dentro del inventario vaya a :

Hosts -> ![plus](img/green_plus.png)

![Ref](img/tower-inventory1.png)

Haga clic en **Save** cuando termine

4. Creación de un proyecto adicional de escaneo

Para que Ansible Tower pueda utilizar los planes de mantenimiento de Insights, debe tener visibilidad para ellos. Cree y ejecute un job/trabajo de escaneo en el inventario utilizando un playbook de escaneo manual, desde la interface de Ansible Tower ir a:

Projects -> ![plus](img/green_plus.png) -> Inventory

Puede utilizar este repositorio de git que contiene los playbooks de escaneo.

https://github.com/ansible/awx-facts-playbooks

![Ref](img/tower-project2.png)


Haga clic en **Save** cuando termine

5. Crear un Trabajo/Job de escaneo/integración para habilitar el botón de Insights en los Hosts

Desde la interface de Ansible Tower ir a:

Templates -> ![plus](img/green_plus.png) -> Job Template

![Ref](img/tower-template1.png)


- Job Type: Seleccionar **Run** 

- Playbook: Seleccione scan_facts.yml

- Credential: Estas credenciales deben coincidir con las de los sistemas operativos del inventario de Insights. Las credencial no tiene que ser una credencial de Insights, deben ser de maquinas **tipo machine**.

* Haga clic para seleccionar Habilitar **Enable Privilege Escalation**  y **Enable Fact Cache** en el campo Opciones.

Haga clic en **Save** cuando termine


> NOTA:
>
> Lo que hace es activar el botón **Insights**  del Host, que es necesario para corregir el inventario de Insights. De lo contrario, el parámetro system_id en el resultado de su trabajo de escaneo se establece en nulo y el botón Insights no aparecerá.

