

# **Design Document: Animatronics and Control System for Emotion-Based Boyfriend Chatbot**

## 1. **Project Overview**

**Objective**: This document focuses on the **technical aspects of the animatronics**, specifically controlling the eye movements and expressions in response to emotional inputs from an AI chatbot. The system uses **STM32** to control servo motors that animate the chatbot's eyes in real time, based on emotions recognized by the chatbot.

---

## 2. **Animatronics System Overview**

The animatronic eye system is designed to physically express emotions such as **happiness, sadness, surprise**, and **anger** through realistic eye movements. These movements are powered by **servo motors**, controlled by an **STM32 microcontroller**, which receives emotional input from the chatbot.

### **Components**:
- **3D Printed Eye Mechanism**: The structure of the eyes, eyelids, and surrounding components are designed for **smooth movement**.
- **Servo Motors**: Each eye uses two **servo motors** to control:
  - **Horizontal (left-right)** movement.
  - **Vertical (up-down)** movement.
  - **Blinking** via a separate servo for the eyelids.
- **STM32 Microcontroller**: Handles the **pulse-width modulation (PWM)** signals required to control the servo motors.
- **Power Supply**: A regulated power source is needed to drive the STM32 and servos, typically **5V** for servos and **3.3V** for the STM32.

---

## 3. **Control System Architecture**

The **control system** is responsible for translating emotional data from the chatbot into physical movements of the animatronics. This section outlines how the **STM32** and **servo motors** work together to achieve this.

### **Key Functions**:
1. **Emotion-to-Movement Mapping**:
   - The chatbot's emotional output (happiness, sadness, surprise, etc.) is mapped to specific **eye movements** and **blinking patterns**.
   - Each emotion triggers a predefined set of movements using the servo motors, controlled by the STM32.

2. **STM32-Based Control**:
   - The **STM32** microcontroller generates **PWM signals** to drive the servo motors based on the received emotional input.
   - The **servo motors** are connected to the STM32’s **GPIO (General-Purpose Input/Output) pins** and controlled through PWM.

---

## 4. **Animatronics Movements**

### **Servo Motor Control**:
The core of the animatronic system relies on **precise servo motor control** for natural eye movements. Here’s how each movement is achieved:

1. **Left-Right (Horizontal Movement)**:
   - A **servo motor** rotates the eye horizontally along the x-axis.
   - The STM32 sends a **PWM signal** to adjust the angle of the servo motor, controlling the position of the eye.

2. **Up-Down (Vertical Movement)**:
   - A second **servo motor** handles vertical movement along the y-axis.
   - By sending different PWM duty cycles, the STM32 adjusts the tilt of the eye to simulate looking up or down.

3. **Blinking**:
   - A separate **servo motor** controls the movement of the eyelids.
   - The eyelid servo is programmed to blink at different speeds and intensities based on the detected emotion (slow blinking for sadness, rapid blinking for surprise, etc.).

---

## 5. **Servo Motor Control using STM32**

### **PWM Signal Generation**:
The STM32 uses **PWM signals** to control the position of each servo motor. The position is determined by the **duty cycle** of the PWM signal:
- **Duty Cycle**: Controls the angle of the servo. For most standard servos:
  - A **1ms pulse** corresponds to 0° (left or up).
  - A **1.5ms pulse** corresponds to the neutral position (center).
  - A **2ms pulse** corresponds to 180° (right or down).

The STM32's **Timer peripherals** are used to generate these PWM signals with precise timing. Here’s how it works:
1. **Configure the Timer**: The STM32's internal timer is set to the appropriate frequency (typically 50Hz for servo control).
2. **Adjust the PWM Duty Cycle**: Depending on the required eye position, the duty cycle of the PWM signal is adjusted in real-time.
3. **Control the Servo**: The STM32 outputs the PWM signal to the GPIO pin connected to the servo motor.

#### **Code Example for PWM Setup**:
```c
// Configure the timer for PWM output
void PWM_Init() {
    TIM_HandleTypeDef htim;
    TIM_OC_InitTypeDef sConfigOC;
    
    htim.Instance = TIMx;  // Choose appropriate timer instance
    htim.Init.Prescaler = 84 - 1;  // Set prescaler
    htim.Init.Period = 19999;  // 20ms period (50Hz)
    HAL_TIM_PWM_Init(&htim);  // Initialize PWM
    
    // Configure PWM channel
    sConfigOC.OCMode = TIM_OCMODE_PWM1;
    sConfigOC.Pulse = 1500;  // 1.5ms pulse width for center position
    HAL_TIM_PWM_ConfigChannel(&htim, &sConfigOC, TIM_CHANNEL_1);
    
    // Start PWM
    HAL_TIM_PWM_Start(&htim, TIM_CHANNEL_1);
}

// Function to set servo angle
void Set_Servo_Angle(uint16_t angle) {
    uint16_t pulse_width = (1000 + (angle * 1000 / 180));  // Map angle to 1-2ms pulse width
    __HAL_TIM_SET_COMPARE(&htim, TIM_CHANNEL_1, pulse_width);
}
```

### **Real-Time Adjustments**:
- The STM32 continuously updates the servo positions based on new emotional data from the chatbot. This allows for **smooth transitions** between eye positions as emotions change in the conversation.

---

## 6. **Communication Between Raspberry Pi and STM32**

The Raspberry Pi handles **emotion recognition** and sends the appropriate movement commands to the STM32 via **UART communication**. Here’s how this communication works:

1. **Emotion Data Transmission**:
   - The Raspberry Pi analyzes the user’s text or speech, recognizes the emotion, and sends a corresponding command to the STM32.
   - Commands include specific values like:
     - Emotion Type (e.g., happiness, sadness)
     - Servo Positions (for each axis: left-right, up-down, blink)

2. **Command Parsing on STM32**:
   - The STM32 receives the command and parses it.
   - Based on the emotion and movement data, the STM32 adjusts the servo motor PWM signals to execute the desired eye movements.

### **UART Communication Example**:
```c
// UART receive callback function
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USARTx) {
        // Parse received command (emotion type and servo positions)
        uint8_t emotion = received_data[0];
        uint16_t horizontal_position = received_data[1];  // Example data
        uint16_t vertical_position = received_data[2];    // Example data

        // Set servo positions based on received data
        Set_Servo_Angle(horizontal_position);
        Set_Servo_Angle(vertical_position);
    }
}
```

---

## 7. **Control Logic for Smooth Movements**

To make the animatronic eye movements look natural, the system uses **interpolation** to smoothly transition between different eye positions.

- **Linear Interpolation**: Smoothly moves the eye from one position to another based on the detected emotion. For example, if the eye needs to move from left to right, it will transition gradually rather than abruptly.

#### **Implementation**:
- The STM32 calculates the intermediate positions and sends continuous PWM updates to the servo motors, ensuring the movement is fluid.

---

## 8. **Error Handling and Calibration**

### **Calibration**:
- The eye servos need to be **calibrated** to ensure the eyes return to a **neutral position** (center) when no emotion is detected.
- Calibration ensures that the servos do not exceed their mechanical limits, preventing damage.

### **Error Handling**:
- If a servo motor fails to respond or moves outside its expected range, the STM32 will trigger a **reset** of the servo to its neutral position.
- If the system detects **communication errors** between the Raspberry Pi and STM32, it will request the data to be resent.

---

## 9. **Future Improvements**

- **Full Facial Animatronics**: Expand the system to control other facial features such as the mouth and eyebrows, adding more expressive capabilities to the chatbot.
- **Improved Control Algorithms**: Implement advanced control algorithms like **PID** (Proportional-Integral-Derivative) control to improve the precision and responsiveness of the servo motors.
- **Head Tracking**: Incorporate a **camera-based head tracking system** so that the chatbot's eyes can follow the user’s movements in real-time.

---

## 10. **Summary**

This design document outlines the **control system** and **animatronics setup** for the emotion-based boyfriend chatbot. The **STM32 microcontroller** plays a key role in controlling the servo motors that animate the chatbot’s eyes in response to emotional input. The system utilizes **PWM control**, **UART communication**,

 and real-time adjustments to ensure smooth, lifelike movements that match the chatbot’s emotional expressions.

