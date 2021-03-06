Примеры кода
===

Python
---

<!-- markdownlint-disable MD031 -->

> **Note** При использовании кириллических символов в кодировке UTF-8 необходимо добавить в начало программы указание кодировки:
> ```python
> # -*- coding: utf-8 -*-
> ```

<!-- markdownlint-enable MD031 -->

### # {#distance}

Функция определения расстояния между двумя точками (**важно**: точки должны быть в одной [системе координат](frames.md)):

```python
import math

# ...

def get_distance(x1, y1, z1, x2, y2, z2):
    return math.sqrt((x1 - x2) ** 2 + (y1 - y2) ** 2 + (z1 - z2) ** 2)
```

### # {#distance-global}

Функция для приблизительного определения расстояния (в метрах) между двумя глобальными координатами (широта/долгота):

```python
import math

# ...

def get_distance_global(lat1, lon1, lat2, lon2):
    return math.hypot(lat1 - lat2, lon1 - lon2) * 1.113195e5
```

### # {#block-takeoff}

Взлет и ожидание окончания взлета:

```python
z = 2  # высота
tolerance = 0.2  # точность проверки высоты (м)

# Запоминаем изначальную точку
start = get_telemetry()

# Взлетаем на 2 м
print navigate(z=z, speed=0.5, frame_id='body', auto_arm=True)

# Ожидаем взлета
while not rospy.is_shutdown():
    # Проверяем текущую высоту
    if start.z + z - get_telemetry().z < tolerance:
        # Взлет завершен
        break
    rospy.sleep(0.2)
```

Вышеприведенный код может быть обернут в функцию:

```python
def takeoff_wait(alt, speed=0.5, tolerance=0.2):
    start = get_telemetry()
    print navigate(z=alt, speed=speed, frame_id='body', auto_arm=True)

    while not rospy.is_shutdown():
        if start.z + alt - get_telemetry().z < tolerance:
            break

        rospy.sleep(0.2)
```

### # {#block-nav}

Лететь в точку и ждать пока коптер долетит в нее:

```python
tolerance = 0.2  # точность проверки прилета (м)
frame_id='aruco_map'

# Летим в точку 1:2:3 в поле ArUco-маркеров
print navigate(frame_id=frame_id, x=1, y=2, z=3, speed=0.5)

# Ждем, пока коптер долетит до запрошенной точки
while not rospy.is_shutdown():
    telem = get_telemetry(frame_id=frame_id)
    # Вычисляем расстояние до заданной точки
    if get_distance(1, 2, 3, telem.x, telem.y, telem.z) < tolerance:
        # Долетели до необходимой точки
        break
    rospy.sleep(0.2)
```

Вышеприведенный код может быть обернут в функцию:

```python
def navigate_wait(x, y, z, speed, frame_id, tolerance=0.2):
    navigate(x=x, y=y, z=z, speed=speed, frame_id=frame_id)

    while not rospy.is_shutdown():
        telem = get_telemetry(frame_id=frame_id)
        if get_distance(x, y, z, telem.x, telem.y, telem.z) < tolerance:
            break
        rospy.sleep(0.2)
```

Более универсальная функция с использованием фрейма `navigate_target`, который совпадает с целевой точкой навигации дрона:

```python
def navigate_wait(x, y, z, speed, frame_id, tolerance=0.2):
    navigate(x=x, y=y, z=z, speed=speed, frame_id=frame_id)

    while not rospy.is_shutdown():
        telem = get_telemetry(frame_id='navigate_target')
        if math.sqrt(telem.x ** 2 + telem.y ** 2 + telem.z ** 2) < tolerance:
            break
        rospy.sleep(0.2)
```

Такой код может быть использован для полета в том числе с использованием фрейма `body`.

### # {#block-land}

Посадка и ожидание окончания посадки:

```python
land()
while get_telemetry().armed:
    rospy.sleep(0.2)
```

Вышеприведенный код может быть обернут в функцию:

```python
def land_wait():
    land()
    while get_telemetry().armed:
        rospy.sleep(0.2)
```

### # {#disarm}

Дизарм коптера (выключение винтов, **коптер упадет**):

```python
# Объявление прокси:
from mavros_msgs.srv import CommandBool
arming = rospy.ServiceProxy('mavros/cmd/arming', CommandBool)

# ...

arming(False)  # дизарм
```

### # {#transform}

Трансформировать позицию (`PoseStamped`) из одной системы координат ([фрейма](frames.md)) в другую, используя [tf2](http://wiki.ros.org/tf2):

```python
import tf2_ros
import tf2_geometry_msgs
from geometry_msgs.msg import PoseStamped

tf_buffer = tf2_ros.Buffer()
tf_listener = tf2_ros.TransformListener(tf_buffer)

# ...

# Создаем объект PoseStamped (либо получаем из топика):
pose = PoseStamped()
pose.header.frame_id = 'map'  # фрейм, в котором задана позиция
pose.header.stamp = rospy.get_rostime()  # момент времени, для которого задана позиция (текущее время)
pose.pose.position.x = 1
pose.pose.position.y = 2
pose.pose.position.z = 3
pose.pose.orientation.w = 1

frame_id = 'base_link'  # целевой фрейм
transform_timeout = rospy.Duration(0.2)  # таймаут ожидания трансформации

# Преобразовываем позицию из старого фрейма в новый:
new_pose = tf_buffer.transform(pose, frame_id, transform_timeout)
```

### # {#upside-down}

Определение, перевернут ли коптер:

```python
PI_2 = math.pi / 2
telem = get_telemetry()

flipped = abs(telem.pitch) > PI_2 or abs(telem.roll) > PI_2
```

### # {#angle-hor}

Расчет общего угла коптера к горизонту:

```python
PI_2 = math.pi / 2
telem = get_telemetry()

flipped = not -PI_2 <= telem.pitch <= PI_2 or not -PI_2 <= telem.roll <= PI_2
angle_to_horizon = math.atan(math.hypot(math.tan(telem.pitch), math.tan(telem.roll)))
if flipped:
    angle_to_horizon = math.pi - angle_to_horizon
```

### # {#circle}

Полет по круговой траектории:

```python
RADIUS = 0.6  # m
SPEED = 0.3  # rad / s

start = get_telemetry()
start_stamp = rospy.get_rostime()

r = rospy.Rate(10)

while not rospy.is_shutdown():
    angle = (rospy.get_rostime() - start_stamp).to_sec() * SPEED
    x = start.x + math.sin(angle) * RADIUS
    y = start.y + math.cos(angle) * RADIUS
    set_position(x=x, y=y, z=start.z)

    r.sleep()
```

### # {#rate}

Повторять действие с частотой 10 Гц:

```python
r = rospy.Rate(10)
while not rospy.is_shutdown():
    # Do anything
    r.sleep()
```

### # {#mavros-sub}

Пример подписки на топики из MAVROS:

```python
from geometry_msgs.msg import PoseStamped, TwistStamped
from sensor_msgs.msg import BatteryState
from mavros_msgs.msg import RCIn

# ...

def pose_update(pose):
    # Обработка новых данных о позиции коптера
    pass

# Остальные функции-обработчики
# ...

rospy.Subscriber('/mavros/local_position/pose', PoseStamped, pose_update)
rospy.Subscriber('/mavros/local_position/velocity', TwistStamped, velocity_update)
rospy.Subscriber('/mavros/battery', BatteryState, battery_update)
rospy.Subscriber('mavros/rc/in', RCIn, rc_callback)
```

Информацию по топикам MAVROS см. по [ссылке](mavros.md).

<!-- markdownlint-disable MD044 -->

### # {#mavlink}

<!-- markdownlint-enable MD044 -->

Пример отправки произвольного [MAVLink-сообщения](mavlink.md) коптеру:

```python
# ...

from mavros_msgs.msg import Mavlink
from mavros import mavlink
from pymavlink import mavutil

# ...

mavlink_pub = rospy.Publisher('mavlink/to', Mavlink, queue_size=1)

# Отправка сообщения HEARTBEAT:

msg = mavutil.mavlink.MAVLink_heartbeat_message(mavutil.mavlink.MAV_TYPE_GCS, 0, 0, 0, 0, 0)
msg.pack(mavutil.mavlink.MAVLink('', 2, 1))
ros_msg = mavlink.convert_to_rosmsg(msg)

mavlink_pub.publish(ros_msg)
```

### # {#rc-sub}

Реакция на переключение режима на пульте радиоуправления (может быть использовано для запуска автономного полета, см. [пример](https://gist.github.com/okalachev/b709f04522d2f9af97e835baedeb806b)):

```python
from mavros_msgs.msg import RCIn

# Вызывается при получении новых данных с пульта
def rc_callback(data):
    # Произвольная реакция на переключение тумблера на пульте
    if data.channels[5] < 1100:
        # ...
        pass
    elif data.channels[5] > 1900:
        # ...
        pass
    else:
        # ...
        pass

# Создаем подписчик на топик с данными с пульта
rospy.Subscriber('mavros/rc/in', RCIn, rc_callback)

rospy.spin()
```

### # {#set_mode}

Сменить [режим полета](modes.md) на произвольный:

```python
from mavros_msgs.srv import SetMode

# ...

set_mode = rospy.ServiceProxy('mavros/set_mode', SetMode)

# ...

set_mode(custom_mode='STABILIZED')
```

### # {#flip}

Флип:

```python
import math

# ...

PI_2 = math.pi / 2

def flip():
    start = get_telemetry()  # memorize starting position

    set_rates(thrust=1)  # bump up
    rospy.sleep(0.2)

    set_rates(pitch_rate=30, thrust=0.2)  # pitch flip
    # set_rates(roll_rate=30, thrust=0.2)  # roll flip

    while True:
        telem = get_telemetry()
        flipped = abs(telem.pitch) > PI_2 or abs(telem.roll) > PI_2
        if flipped:
            break

    rospy.loginfo('finish flip')
    set_position(x=start.x, y=start.y, z=start.z, yaw=start.yaw)  # finish flip

print navigate(z=2, speed=1, frame_id='body', auto_arm=True)  # take off
rospy.sleep(10)

rospy.loginfo('flip')
flip()
```

Необходимо использование [специальной сборки PX4 для Клевера](firmware.md#прошивка-для-клевера). Перед выполнением флипа необходимо принять все меры безопасности.

### # {#calibrate-gyro}

Произвести калибровку гироскопа:

```python
from pymavlink import mavutil
from mavros_msgs.srv import CommandLong
from mavros_msgs.msg import State

# ...

send_command = rospy.ServiceProxy('/mavros/cmd/command', CommandLong)

def calibrate_gyro():
    rospy.loginfo('Calibrate gyro')
    if not send_command(command=mavutil.mavlink.MAV_CMD_PREFLIGHT_CALIBRATION, param1=1).success:
        return False

    calibrating = False
    while not rospy.is_shutdown():
        state = rospy.wait_for_message('mavros/state', State)
        if state.system_status == mavutil.mavlink.MAV_STATE_CALIBRATING or state.system_status == mavutil.mavlink.MAV_STATE_UNINIT:
            calibrating = True
        elif calibrating and state.system_status == mavutil.mavlink.MAV_STATE_STANDBY:
            rospy.loginfo('Calibrating finished')
            return True

calibrate_gyro()
```

> **Note** В процессе калибровки гироскопов дрон нельзя двигать.
