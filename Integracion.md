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

Credentials -> Adicionar credenciales (+)

En el campo **Tipo de credencial**, ingrese Insights o haga clic en el botón y selecciónelo en la ventana emergente del tipo de credencial.

![Ref](img/tower-creds.png)

y rellene los campos con los mismos datos de Registro de la maquinas y los del portal **Red Hat Customer Portal**

Haga clic en **Save** cuando termine


2. Crear un proyecto de integración con Red Hat Insights, desde la interface de Ansible Tower ir a:

Projects -> Adicionar proyecto (+)

El campo **Credential** se rellena previamente con las credenciales de Insights que creó anteriormente. De lo contrario, ingrese la credencial o haga clic en el botón y selecciónelo en la ventana emergente.

![Ref](img/tower-project.png)

Haga clic en **Save** cuando termine

