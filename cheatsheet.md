# vtysh

## Mostrar

    show interface

<br>

    show interface brief


# Chequeo configuración RIP - vtysh

## Rutas estáticas

```
show ip route static
```

## Configuración RIP

```
show ip rip
```

```
show running-config
```

## Base de datos RIP

```
show ip rip database
```

## Vecinos RIP

```
show ip rip neighbor
```

## Tabla de ruteo completa

```
show ip route
```

# Eliminar  configuraciones
    configure terminal

    interface eth0
        no description
        no ip address 192.168.10.1/23
        shutdown
    exit

    end
    write memory

## Activar terminal de configuracion

    configure terminal
    ...
    ...
    ...
    end

## Guardar cambios

    write


