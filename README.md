# Anti-theft-system-

To many people, a motor vehicle is often the second largest investment they make during their lifetime, only behind a home or land. Vehicle theft in the form of hijacking and carjacking is a prevalent issue in many countries, such as the USA, New Zealand and South Africa. Carjacking is described as the theft of stationery cars that don't contain any passengers. Hijacking is defined as the theft of a vehicle with occupants. Statistics indicate that there are approximately 88,000 vehicles stolen in South Africa alone during 2019. This is concerning as oftentimes the vehicle's driver's life is put at risk as they are forced to step out of their vehicle at gun point. There are a few commercial solutions that try to tackle the carjacking aspect of vehicle theft by attaching passive immobilizers in a hidden spot of the vehicle. These immobilizers require the driver to deactivate them within a certain time period after the engine's ignition to prevent the vehicle from stalling. However, this solution is not effective in cases of hijacking as the immobilizer would already be deactivated by the driver. The prototype discussed in this wiki aims to be effective in both forms of vehicle theft.

## Overview

![Screen Shot 2021-04-14 at 6 23 49 PM](https://user-images.githubusercontent.com/43019063/114787933-93589c80-9d4e-11eb-94a2-649db81f9e6e.png)
**Figure 1.** A block diagram laying out the fundamental hardware components of the device and how the device interfaces with the vehicle.

The prototype makes use of a Raspberry Pi 3b+ for computerized control, a GSM shield (with a SIM card from a valid network provider in the country of use) to read and send text messages and a GPS shield to read world co-ordinates. The prototype device communicates with the user’s cellular device using text messaging. The text messages are then parsed, and the necessary action will be taken. 

The Raspberry Pi interfaces with the vehicle by means of the ignition 50 (IGN 50) wire. IGN 50 is a wire that connects the vehicle’s fuse box to the vehicle’s Engine Control Unit (ECU). The Raspberry Pi introduces a digital switch in this wire. 

## How the device works

When the vehicle is first turned on, the IGN 50 input is read by the Pi. The Pi then sends a text message notification to the user’s mobile device, alerting them that their vehicle is turned on. If the vehicle is being used by someone that is known by the owner, the owner may choose to do nothing after receiving the notification. But if the vehicle has been turned on without the owner’s knowledge, the owner may choose to immobilize the vehicle with the appropriate text message command. The GSM shield will receive the notification and will cause an interrupt in the Pi’s main thread. The Pi then queries the GSM shield for the notification received and will parse this notification to read the actual command. After reading the command the Pi will open the digital switch between the fuse box and ECU. The ECU then has no knowledge of the fuse box’s status and begins to assume a worst-case scenario. The ECU will then send a message to the engine block to kill all processes. This turns the vehicle off. 

If the owner wishes to remobilize the vehicle, the owner may do so by sending the appropriate text message command. The Pi will parse this message (as discussed previously) and will close the digital switch. At this point, the driver can switch the ignition on and the vehicle will behave normally.

If the owner wishes to know where the vehicle is, the owner may do so sending the appropriate text message command. The Pi will parse this message and will query the GPS shield for the current coordinates of the vehicle. The Pi will then create a message payload with the coordinates and send it to the GSM shield. The shield will then send this payload back to the owner via text message, and then owner may view the coordinates from his mobile device. 

Before any incoming command is parsed, the phone number it is received from is checked for authentication. This prevents random people taking control over your vehicle. The Pi does this authentication by checking the phone number of the received text message against a file of registered numbers. If the number exists in the file, the command will be parsed. If not, the message will be discarded. The owner may also control which members have access to the device by registering new or deleting existing phone numbers. These commands are also carried out with the appropriate text message. When registering a new number, the Pi will check if the number already exists in its database to prevent duplicate registrations. If the number exists, the Pi will ignore the command. Otherwise, it will add the number to its existing database. The Pi also does a similar check when deleting. If the number does not exist in the database, the command will be ignored. Otherwise, the number will be removed from the database and that particular user will no longer be able to control the immobilization or tracking technologies. 

![Screen Shot 2021-04-14 at 7 12 54 PM](https://user-images.githubusercontent.com/43019063/114791970-6d82c600-9d55-11eb-8236-013ff414c318.png)

**Figure 2.** A list of accepted commands by the device and the action taken for each command.

## Description of important functions

**1. Event Handling**

```
def event_handling(registered_numbers):
    """ Handling commands received through SMS """
    screen = serial.Serial("/dev/ttyUSB0",9600,timeout=3) # Reading serial port to check for any notifications
    notification = []
    notification = GSM_status(screen,80).split(",")
    if notification[0] != '' and "+CMT" in notification[0]:
        print("new sms received")
        message,number = read_message(notification)
        if user_verification(number,registered_numbers):
            if message == "START": # Command to restart a stalled car
                turn_on()
                print("LED turned on")
            elif message ==  "STOP": # Command to stall a running car
                turn_off()
                print("LED turned off")
            elif "REGISTER DEVICE:" in message: #Command to register a new user to  database of registered users
                print("Registering")
                new_user = user_number(message)
                # Checking if number to be added already exists in the file, and only adding if it is not 
                if new_user in registered_numbers:
                    print("Number already stored")
                else:
                    registered_numbers.append(new_user)
                    print("Adding number")
            elif "DELETE DEVICE:" in message:   #Command to remove a user from database of registered users
                del_user = user_number(message)
                # Checking if number to be deleted is stored in the file, and only deleting if it is
                if del_user in registered_numbers:
                    i = registered_numbers.index(del_user)
                    del registered_numbers[i]
                    print("User deregistered")
                else:
                    print("User does not exist in database")
            elif "LOCATION" in message:
                send_coordinates(number)
```


**2. Parsing commands** 

```
def read_message(notification):
    param1 = notification[0]
    param2 = notification[3]
    pos1 = param1.find('"')
    number = param1[pos1+2:-1]
    pos2 = param2.find('\n')
    message = param2[pos2+1:-2]
    return (message.upper(), int(number))
```

**3. Authentication**

```
def user_verification(number,registered_numbers):
    """ Verifying if command is from a registered user """
    if number in registered_numbers:return True
    else: return False
```

## Further Development

A Raspberry Pi is not a true embedded device as it runs an operating system (OS), i.e., the Linux based Raspbian OS. Devices used in real time systems do not run an OS as it causes a significant delay in how the device works. Before this prototype can be commercialized, it must be redeveloped on a bare-metal embedded board with software written in C. This will allow for a real time system that can parse and carry out commands much faster than the prototype, providing better control over the vehicle. 

Furthermore, a mobile application will also need to be developed to provide the users with a friendly and efficient interface to communicate with the device. The method of communication between the mobile device and the embedded device will need to change from text messaging to cloud-based communication using MQTT or HTTPS protocols. This will allow easy integration into the mobile app, enable secure data transmission and increase the rate at which the mobile device and the embedded device can communicate. The mobile app would also implement a map interface that uses the received coordinates (from the tracking command) to quickly and easily visualize the location of the vehicle.
