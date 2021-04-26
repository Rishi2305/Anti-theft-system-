# Anti-theft-system-

## Contents

1. [Introduction](https://github.com/Rishi2305/Anti-theft-system-/blob/main/README.md#introduction)
2. [Overview](https://github.com/Rishi2305/Anti-theft-system-/blob/main/README.md#overview)
3. [How the device works](https://github.com/Rishi2305/Anti-theft-system-/blob/main/README.md#how-the-device-works)
4. [Description of important functions](https://github.com/Rishi2305/Anti-theft-system-/blob/main/README.md#description-of-important-functions)
5. [Further development](https://github.com/Rishi2305/Anti-theft-system-/blob/main/README.md#further-development)
6. [Conclusion](https://github.com/Rishi2305/Anti-theft-system-/blob/main/README.md#conclusion)
7. [References](https://github.com/Rishi2305/Anti-theft-system-/blob/main/README.md#references)

## Introduction

To many people, a motor vehicle is often the second largest investment they make during their lifetime, only behind a home or land. Vehicle theft in the form of hijacking and carjacking is a prevalent issue in many countries, such as the USA, New Zealand and South Africa. Carjacking is described as the theft of stationary cars that don't contain any passengers. Hijacking is defined as the theft of a vehicle with occupants. Statistics indicate that there are approximately 88,000 vehicles stolen in South Africa alone during 2019. This is concerning as oftentimes the vehicle's driver's life is put at risk as they are forced to step out of their vehicle at gun point. There are a few commercial solutions that try to tackle the carjacking aspect of vehicle theft by attaching passive immobilizers in a hidden spot of the vehicle. These immobilizers require the driver to deactivate them within a certain time period after the engine's ignition to prevent the vehicle from stalling. However, this solution is not effective in cases of hijacking as the immobilizer would already be deactivated by the driver. The prototype discussed in this wiki aims to be effective in both forms of vehicle theft.

## Overview

![Screen Shot 2021-04-14 at 6 23 49 PM](https://user-images.githubusercontent.com/43019063/114787933-93589c80-9d4e-11eb-94a2-649db81f9e6e.png)
**Figure 1.** A block diagram laying out the fundamental hardware components of the device and how the device interfaces with the vehicle.

The prototype discussed in this wiki was developed using Python3 for software, a Raspberry Pi 3b+ for computerized control, a GSM shield (with a SIM card from a valid network provider in the country of use) to read and send text messages and a GPS shield to read world co-ordinates. The prototype device communicates with the user’s cellular device using text messaging. The text messages are then parsed, and the necessary action will be taken. 

The Raspberry Pi interfaces with the vehicle by means of the ignition 50 (IGN 50) wire. IGN 50 is a wire that connects the vehicle’s fuse box to the vehicle’s Engine Control Unit (ECU). The Raspberry Pi introduces a digital switch in this wire. When the switch is closed, the vehicle will drive normally, and when the switch is opened, the vehicle will stall and come to a stop gradually. This gives the user the ability to control the engine status from their mobile device.

## How the device works

When the vehicle is first turned on, the IGN 50 input is read by the Pi. The Pi then sends a text message notification to the user’s mobile device, alerting them that their vehicle is turned on. If the vehicle is being used by someone that is known by the owner, the owner may choose to do nothing after receiving the notification. But if the vehicle has been turned on without the owner’s knowledge, the owner may choose to immobilize the vehicle with the appropriate text message command. The GSM shield will receive the notification and will cause an interrupt in the Pi’s main thread. The Pi then queries the GSM shield for the notification received and will parse this notification to read the actual command. After reading the command the Pi will open the digital switch between the fuse box and ECU. The ECU then has no knowledge of the fuse box’s status and begins to assume a worst-case scenario. The ECU will then send a message to the engine block to kill all processes. This turns the vehicle off. 

If the owner wishes to remobilize the vehicle, the owner may do so by sending the appropriate text message command. The Pi will parse this message (as discussed previously) and will close the digital switch. At this point, the driver can switch the ignition on and the vehicle will behave normally.

If the owner wishes to know where the vehicle is, the owner may do so by sending the appropriate text message command. The Pi will parse this message and will query the GPS shield for the current coordinates of the vehicle. The Pi will then create a message payload with the coordinates and send it to the GSM shield. The shield will then send this payload back to the owner via text message, and the owner may view the coordinates from his mobile device.

Before any incoming command is parsed, the phone number it is received from is checked for authentication. This prevents random people from taking control over the vehicle. The Pi does this authentication by checking the phone number of the received text message against a file of registered numbers. If the number exists in the file, the command will be parsed. If not, the message will be discarded. The owner may also control which members have access to the device by registering new or deleting existing phone numbers. These commands are also carried out with the appropriate text message. When registering a new number, the Pi will check if the number already exists in its database to prevent duplicate registrations. If the number exists, the Pi will ignore the command. Otherwise, it will add the number to its existing database. The Pi also does a similar check when deleting. If the number does not exist in the database, the command will be ignored. Otherwise, the number will be removed from the database and that particular user will no longer be able to control the immobilization or tracking technologies. 

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
        print("new payload received")
        message,number = read_message(notification)
        if user_verification(number,registered_numbers):
            if message == "START": # Command to restart a stalled car
                turn_on()
                print("Vehicle turned on")
            elif message ==  "STOP": # Command to stall a running car
                turn_off()
                print("Vehicle turned off")
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

This function is the main event handler and handles all the interrupts that arise. It loops infinitely within the main function and checks the GSM shield for notifications every 3 seconds. If a notification is received, the payload is parsed to retrieve the message and the number of the sender. The number is used to authenticate the sender. If the sender is not authenticated, the message is discarded. If the sender is authenticated, the appropriate action is taken based on the command.

**2. Parsing commands** 

```
def read_message(notification):
    """ Parsing notification to retreive message """
    param1 = notification[0]
    param2 = notification[3]
    pos1 = param1.find('"')
    number = param1[pos1+2:-1]
    pos2 = param2.find('\n')
    message = param2[pos2+1:-2]
    return (message.upper(), int(number))
```

When a new notification is received, there are a few steps that must be taken before the actual message and number of the sender can be accessed. The payload contains many newline and carriage return characters and other pieces of data that do not hold significance in the workings of this prototype, such as the name of the service provider, the number of messages in the device’s inbox, and the date and time of the message received. The function above accurately navigates the received payload string to discard the insignificant data and retrieve only the content of the message and the cellular number that the message was sent from. It then returns the data in the form of a tuple. This tuple is parsed in the event handler and the necessary action is taken.

**3. Authentication**

```
def user_verification(number,registered_numbers):
    """ Verifying if command is from a registered user """
    if number in registered_numbers:return True
    else: return False
```

This function is responsible for authenticating the sender of the command. The parameters of the function are the number of the sender and a list of authorizing numbers. The function returns True if the number is authorized and False otherwise. The list of authorized numbers is retrieved from a text file stored on the Pi’s memory. The file would be read in once, in the main loop, each time the vehicle is turned on and the data is stored in a list. This is to make the device efficient by not wasting time opening and reading a file once per loop. If a new number is registered to the device or an existing number is deleted, the list containing the authorized numbers is immediately updated. The text file will only be updated once when the vehicle is turned off. This is also to the benefit of the device’s efficiency. 

## Further Development

A Raspberry Pi is not a true embedded device as it runs an operating system (OS), i.e., the Linux based Raspbian OS. Devices used in real time systems do not run an OS as it causes a significant delay in how the device works. Before this prototype can be commercialized, it must be redeveloped on a bare-metal embedded board with software written refactored in C. This will allow for a real time system that can parse and carry out commands much faster than the prototype, providing better control over the vehicle. 

Furthermore, a mobile application will also need to be developed to provide the users with a friendly and efficient interface to communicate with the device. The method of communication between the mobile device and the embedded device will need to change from text messaging to cloud-based communication using MQTT or HTTPS protocols. This will allow easy integration into the mobile app, enable secure data transmission and increase the rate at which the mobile device and the embedded device can communicate. The mobile app would also implement a map interface that uses the received coordinates (from the tracking command) to plot the vehicle's position on a visual map. The map interfcaewould need to run in real time, constantly updating the vehicle's coordinates in the visual map. This would enable the user to quickly and easily visualize the location of the vehicle. 

## Conclusion

## References
