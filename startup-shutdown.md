# Стартиране на Нестум след тотален shutdown

1. Стартира се (физически) FS01 (Fujitsu файлов сървър)
2. Стартират се (физически) SN01, SN02 (сервизните възли)
3. Стартиране на останалите машини:
   
    1. Преподготовка
    След успешното стартиране на горните ще има достъп до FE01.

    ```bash
    ssh sysadm@hpc-lab.sofiatech.bg
    systemctl restart opt.mount
    ssh sn01 -p 666
    systemctl restart opt.mount
    ```
   
    2. Стартиране на файлов сървър 2 - FS02 (Supermicro)
    След горните команди трябва да се намираме на сервизен възел 1 и да имаме възможност да стартираме останалите машини.

    ```bash
    ipmitool -H da02.irmc -U admin -P admin power on
    ipmitool -H fs02.irmc -U admin -P admin power on
    ```
    
    3. Стартиране на изчислителни възли:
    ```bash
    /home/sysadm/admin/power.sh 1-24 on
    ```

4. Поправка на счупени сървиси:
    Целият контрол става през sn01
   
    1. Рестартиране на munge/slurm
    ```bash
    ansible -m shell -a 'systemctl restart munge' cn0*
    ansible -m shell -a 'systemctl restart slurm' cn0*
    ```
   
    2. Рестартиране на главния възел sn02
    Понякога slurm контролера не успява да тръгне както трябва.
    За целта: 
    ```bash
    ssh sn02 
    sudo reboot now
    ```
    !!!Внимание След успешното стартиране на sn02 да се повтори 4.1

5. Проблеми с маунтване на потребителски директории във fs02:
    Понякога zfs sharefs се сетва на off когато се спира тотално fs02.
    
    
    1. Поправка на Network shares
    За да поправите потребителски директории от него, които не се маунтват от него:
    ```bash
    ssh fs02
    zfs get sharenfs 
    ```
    Резултата от горната команда трябва да изглежда по следния начин:
    ```bash
    sysadm@fs02:~$ sudo zfs get sharenfs
    NAME                          PROPERTY  VALUE            SOURCE
    zlocal                        sharenfs  off              default
    znfs                          sharenfs  off              default
    znfs/export                   sharenfs  on               local
    znfs/export/home              sharenfs  on               local
    znfs/export/home/inf          sharenfs  on               inherited from znfs/export/home
    znfs/export/home/smgrobotics  sharenfs  on               inherited from znfs/export/home
    znfs/export/work              sharenfs  rw=@10.2.0.0/15  local
    znfs/export/work/inf          sharenfs  rw=@10.2.0.0/15  inherited from znfs/export/work
    znfs/export/work/mc           sharenfs  rw=@10.2.0.0/15  inherited from znfs/export/work
    znfs/export/work/smgrobotics  sharenfs  rw=@10.2.0.0/15  inherited from znfs/export/work
    ```
    Ако не е като горното, напр. `znfs/export` е `off`, използвайки `zfs set sharenfs=on znfs/export` или аналогично, доведете състоянието до горното.

    
    2. Рестартиране на autofs:
    ```bash
    ssh fe001
    sudo systemctl restart autofs.service
    ```
    ```bash
    ssh sn01
    ansible -m shell -a 'systemctl restart autofs.service' cn0*
    ```
    
    Горните две би следвало да решат проблема.

    Възможно е да се наложи рестартиране и на други конкретни сървиси! Горното е ориентировъчно за най-основните проблеми.

---

# Тотален/Частичен Shutdown:


1. Изключване на изчислителните възли:
Най-безопасно и лесно е изключването на изчислителни възли
    ```bash
    ssh sysadm@hpc-lab.sofiatech.bg
    systemctl restart opt.mount
    ssh sn01 -p 666
    /home/sysadm/admin/power.sh 1-24 off
    ```
    За по-кратки прекъсвания (< 2 часа) това е напълно достатъчно. Зарядът на 100% зареден UPS ще е напълно достатъчен, за да се "преживее" спирането.  При включването им може да се наложи рестартиране на sn02 и на munge/slurm - вж 4.1, 4.2.


2. Изключване на файловите сървъри:

    1. FS02(Supermicro)
    ```bash
        ssh sn01 -p 666
        ipmitool -H fs02.irmc -U admin -P admin chassis power soft
        ipmitool -H da02.irmc -U admin -P admin chassis power soft
    ```
    2. FS01(Fujitsu)
    ```bash
        ssh sn01 -p 666
        ipmitool -H fs01.irmc -U admin -P admin chassis power soft
    ```

3. Изключване на сервизните възли:
    ```bash
        ssh sn01 -p 666
        ipmitool -H sn02.irmc -U admin -P admin chassis power off
        sudo shutdown now
    ```
В края на стъпка 3, целият Нестум е изключен.