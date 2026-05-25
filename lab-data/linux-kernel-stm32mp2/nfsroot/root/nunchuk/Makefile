ifneq ($(KERNELRELEASE),)
obj-m := nunchuk.o
else
KDIR := $(HOME)/linux-kernel-stm32mp2-labs/src/linux
all:
	$(MAKE) -C $(KDIR) M=$$PWD

clean:
	$(MAKE) -C $(KDIR) M=$$PWD clean
endif
