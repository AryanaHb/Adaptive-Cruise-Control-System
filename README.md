%Program for implementing adaptive cruise control
close all;
clear;
clc;
% Let the fun begin by connecting to Arduino
arduino_board = arduino('COM9', 'Uno', 'Libraries', {'Ultrasonic','ExampleLCD/LCDAddon'});
myLcd =
addon(arduino_board,'ExampleLCD/LCDAddon','RegisterSelectPin','D7','EnablePin','D6','DataPin
s',{'D5','D4','D3','D2'});
startLcd(myLcd);
printLCD(myLcd,char("Welcome!"));
pause(2);
printLCD(myLcd,char("Group #5"));
pause(2);
printLCD(myLcd,char("Shabnam"));
pause(2);
printLCD(myLcd,char("Soror"));
pause(2);
printLCD(myLcd,char("Aryana"));
pause(2);
% Here lies the pin numbers for the btns
btnSpeedCruise = 'A0';
btnSpeedAdaptive = 'A1';
btnCncl = 'A2';
btnSpeedInc = 'A3';
btnSpeedDec = 'A4';
btnVol = 4;
% Here lies the pin numbers for the ultrasonic sensor
usTrig = 'D8';
usEcho = 'D9';
% Setup button pins
configurePin(arduino_board, btnSpeedCruise, 'DigitalInput');
configurePin(arduino_board, btnSpeedAdaptive, 'DigitalInput');
configurePin(arduino_board, btnCncl, 'DigitalInput');
configurePin(arduino_board, btnSpeedInc, 'DigitalInput');
configurePin(arduino_board, btnSpeedDec, 'DigitalInput');
% how close is fine
distanceArray = 0:0.1:0.6;
distanceGap = 0.1;
blink = 0.5;
noActionSecond = 1;
% Init ultrasonic sensor
uSensor = ultrasonic(arduino_board, usTrig, usEcho);
% Init vals
velocity = 0;
isCCActive = false;
isACCActive = false;
% LCD Val to show
isLcdOn = true;
noActionSecondPassed = tic;
velocityFixed = 0;
% While we're here, we're here
while true
% Is a btn pressed? if yes then do what is written
if readVoltage(arduino_board, btnSpeedCruise) >= btnVol
% you're pushing the speed set btn
isCCActive = true;
noActionSecondPassed = tic;
disp('btnSpeedCruise is pressed');
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
elseif readVoltage(arduino_board, btnSpeedAdaptive) >= btnVol
% you're pushing the ACC btn
isACCActive = true;
isCCActive = true;
velocityFixed = velocity;
noActionSecondPassed = tic;
disp('btnSpeedAdaptive is pressed');
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
elseif readVoltage(arduino_board, btnCncl) >= btnVol
% you're pushing the cancel btn
isCCActive = false;
isACCActive = false;
velocityFixed = 0;
noActionSecondPassed = tic;
disp('btnCncl is pressed');
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
elseif readVoltage(arduino_board, btnSpeedInc) >= btnVol
% you're going faster
if ~isACCActive
velocity = velocity + 1;
end
noActionSecondPassed = tic;
disp('btnSpeedInc is pressed');
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
elseif readVoltage(arduino_board, btnSpeedDec) >= btnVol
% you're slowing down
if ~isACCActive
velocity = max(0, velocity - 1);
end
noActionSecondPassed = tic;
disp('btnSpeedDec is pressed')
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
end
% ACC part of the code
if isACCActive
distanceCurrent = readDistance(uSensor);
for k = 1:length(distanceArray)
if(distanceCurrent < 0.04)
% we crashed
velocity = 0;
showLcd(myLcd, velocity, isCCActive, isACCActive, distanceCurrent);
elseif(distanceArray(k) < distanceCurrent && distanceCurrent < distanceArray(k) +
distanceGap)
switch k
case 1
velocity = max(0, velocity - 5);
case 2
velocity = max(0, velocity - 4);
case 3
velocity = max(0, velocity - 3);
case 4
velocity = max(0, velocity - 2);
case 5
velocity = max(0, velocity - 1);
end
end
end
if(distanceCurrent > 0.04)
if velocity < velocityFixed
velocity = velocity + 1;
end
end
% Blinking effect for LCD
if isLcdOn
showLcd(myLcd, velocity, isCCActive, isACCActive, distanceCurrent);
isLcdOn = false;
else
clearLCD(myLcd);
isLcdOn = true;
end
pause(blink);
else
if ~isLcdOn
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
isLcdOn = true;
end
end
% slow down logic
elapsed_time_button_press = toc(noActionSecondPassed);
if elapsed_time_button_press >= noActionSecond && ~isCCActive && ~isACCActive
velocity = max(0, velocity - 0.5);
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
noActionSecondPassed = tic;
end
% Update LCD
if ~isACCActive || isLcdOn
showLcd(myLcd, velocity, isCCActive, isACCActive, 1);
end
pause(0.1);
end
function startLcd(lcd)
initializeLCD(lcd, 'Rows', 2, 'Columns', 16);
printLCD(lcd, 'Speed: 0');
end
function showLcd(lcd, velocity, cc, acc, distance)
clearLCD(lcd);
if cc && ~acc
mode = 'Cruise ON';
elseif acc
mode = 'Adaptive ON';
else
mode = 'Manual';
end
if distance < 0.04
printLCD(lcd, 'Crashed');
else
printLCD(lcd, ['Speed: ' num2str(velocity)]);
printLCD(lcd, mode);
end
end
