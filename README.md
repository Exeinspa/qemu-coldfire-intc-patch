# QEMU  interrupt controller patch for priority
Emulating hardware is useful to let you make a software intended for a different platform run on a different machine. QEMU is a piece of software supporting many target architectures, among others the M68k cores. Inspecting the M68k implementation, you may come to realize, that QEMU for the M68k based SoCs is not complete. Instead, they implemented only the minimum required for the Linux Kernel to run. Specialized instructions like the "movec"  with target FLASHBAR and RAMBAR1 used only in early SoC initialization, are not implemented, and also some peripherals common in these SoCs are missing. Use the M68k implementation of the QEMU to test Coldfire based hardware can be not a trivial task.

# The Interrupt Controller is buggy in handling the priority
There are 8 levels of interrupts, starting at 0 and ending at 7. They are organized such as Level 0 interrupts are strictly low priority stuff,  whereas Level 7 has the interrupts that Really Ought To Be Dealt With Right Now. The core M68k then ignores any interrupts equal to or below your current IPL level. The coldfire v2 core SOC use two stage priority mechanism, first all 63 interrupt source are organized into 7 levels with the level 7 holds highest priority, and within one level, can support up to 9 priorities. 
To support the interrupt architecture of the **68K/ColdFire** programming model, the combined **63 interrupt sources** are organized as **7 levels**, with each level supporting up to **9 prioritized requests**. Consider the interrupt priority structure shown in table, which orders the interrupt levels/priorities from highest to lowest.

|interrupt Level ICR[IL]|Priority ICR[IP]   |Supported Interrupt Sources|
|-----------------------|-------------------|---------------------------|
| 7                     | 7-5<br>mid<br>3-0 |#8-63<br>7 (IRQ7)<br>#8-63 |
| 6                     | 7-5<br>mid<br>3-0 |#8-63<br>6 (IRQ6)<br>#8-63 |
| 5                     | 7-5<br>mid<br>3-0 |#8-63<br>5 (IRQ5)<br>#8-63 |
| 4                     | 7-5<br>mid<br>3-0 |#8-63<br>4 (IRQ4)<br>#8-63 |
| 3                     | 7-5<br>mid<br>3-0 |#8-63<br>3 (IRQ3)<br>#8-63 |
| 2                     | 7-5<br>mid<br>3-0 |#8-63<br>2 (IRQ2)<br>#8-63 |
| 1                     | 7-5<br>mid<br>3-0 |#8-63<br>1 (IRQ1)<br>#8-63 |
The level and priority is fully programmable for all sources except interrupt sources 1–7. Interrupt source 1–7 (from the Edge port module) are fixed at the corresponding level’s midpoint priority. Thus, a maximum of 8 fully-programmable interrupt sources are mapped into a single interrupt level.
The “fixed” interrupt source is hardwired to the given level, and represents the mid-point of the priority within the level. For the fully-programmable interrupt sources, the 3-bit level and the 3-bit priority within the level are defined in the 8-bit interrupt control register (ICRnx).
It seems like that the Linux kernel for the M68k target won't use the level+priority mechanism, and that it just threat the interrupt as the old m68k core does. This lead some malfunctions, because QEMU assumes the software to be Linux and it copies the data within the INTC registers into the M68k core Status Register (SR).

Here comes a problem because loading SR with incoherent values lead interrupts not to happen.


