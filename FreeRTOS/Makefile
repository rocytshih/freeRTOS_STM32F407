#######################################
# CFLAGS
#######################################
# cpu
CPU = -mcpu=cortex-m4

# fpu
FPU = -mfpu=fpv4-sp-d16

# float-abi
FLOAT-ABI = -mfloat-abi=hard

# mcu
MCU = $(CPU) -mthumb $(FPU) $(FLOAT-ABI)

# C includes
C_INCLUDES =  \
-Iinclude

# optimization
OPT = -Og

PREFIX = arm-none-eabi-
CFLAGS = $(C_INCLUDES) $(OPT) -Wall -fdata-sections -ffunction-sections

CC = gcc

# SRC_OBJ = $(patsubst %.c, %.o, $(wildcard *.c))  
SRC_FILES = $(wildcard src/*.c)  
SRC_OBJ = $(SRC_FILES:.c=.o)  
SRC_LIB = libfreeRTOS.a

libfreeRTOS : $(SRC_LIB)

$(SRC_LIB) : $(SRC_OBJ)
	$(AR) rcs $@ $^
	
clean :
	$(RM) $(SRC_OBJ) $(SRC_LIB)