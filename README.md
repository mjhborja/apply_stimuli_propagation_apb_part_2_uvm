# Applying Stimuli Propagation to a Design - Part 2
## Continuation
From Part 1 [[1](https://github.com/mjhborja/apply_stimuli_propagation_apb_part_1_uvm)], our protagonist received three assets that his / her manager provided in relation to a task, which was to exercise a design. Without knowledge whether the VIP included does the job or not, we discovered that it actually produces a waveform dump after running a simulation. 

Warning: If you already decided you are better off writing the test bench from scratch, then this post is not for you.

Since you are still reading, that means you are interested to learn something as we go along with what this post has to offer.

## What will you learn here?
Evaluating a compiling and running VIP might just save us man hours debugging our own work. For one, your manager will not have given the VIP if it is not of any use in this task. The key to maximize reuse of existing assets is making minimal changes just to make sure it does what it is supposed to do.

### What do we know?
Currently, we understand how the APB 3-signal handshake works. And we know this is the basis for the model assessment we will perform given the information produced by the simulation namely, a simulation log file and waveform value change dump.

### What's left for you to do?
At this point, the engineer might need to learn the phases a UVM simulation goes through and how it maps out to those two pieces of information prior to diving into the contents of either. From there, we will then interpret what the waveform means and if it reflects the correct behavior for both the VIP stimuli and DUV response.

## UVM phases & observable behavior
There are 9 common phases and 12 run-time phases included in the UVM specification [[4](https://www.accellera.org/images/downloads/standards/uvm/UVM_Class_Reference_Manual_1.2.pdf)]. And [[6](https://www.chipverify.com/uvm/uvm-phases)] has a very good discussion of how these phases are utilized as a synchronizing mechanism of a UVM simulation.

![diagram_004 1-uvm_phases](https://www.chipverify.com/images/uvm/uvm-phases.png)
\
Diagram courtesy of [[6](https://www.chipverify.com/uvm/uvm-phases)]

While the print outs in all phases may be reflected in a log file, only the activity of nets during run-time phases are included in the value change dump. 

Since UVM is a discrete event type of simulation, the unifying element in both files are the timestamps. To map the events occuring on the waveform, you may match the design hierarchy from the "Get Signals to Display" dialog of the EPWave waveform viewer app with the instance name of the DUV in the top.sv source code. 

![diagram_003 19-alteration_9_c](https://user-images.githubusercontent.com/50364461/213841020-d3c1cd8f-e5b2-4b42-982e-ce5895ee2b42.png)

You will recognize that ".dut_slave" is the hierarchical instance name under "top" as shown in the diagram above. It reflects the design hierarchy in the top.sv file where "dut_slave" is the instance name of the design "apb_slave" from line 14. And, this instance occurs within the "top" module as shown in line 9 of top.sv.
![diagram_004 7-stimuli_apb_p2_source_code](https://user-images.githubusercontent.com/50364461/213971484-2a49a645-b625-45ec-a106-6c11d768e0c6.png)

Another comparison that you can perform is between the signals of the "dut_slave" instance in lines 15 through 23 and the signals on the waveform.
![diagram_003 21-alteration_10](https://user-images.githubusercontent.com/50364461/213964402-8bd6a6d6-77c0-4631-a7e6-45fd624c516b.png)

Finally, you will want to check the connection of the DUV with the VIP test bench as discussed in [[7](https://github.com/mjhborja/hello_world_uvm)] under the "Connection between top and the UVM test bench", "UVM Build phase" and "UVM Connect phase" sections. There is a slight variation in terms of invoking the "run_test()" from "uvm_root" and the "set" method for the uvm_config_db. For the APB VIP, this is located in a program block in the pgm_test.sv file instead of the test bench top module directly.

Now, you have established the connection between the VIP stimuli and the DUV. You may or may not check the internals of the driver and sequence. To keep effort low, it is recommended you keep it black box. So, don't.

Instead, let us proceed with the behavior of the VIP and the DUV that can be observed from the waveform.

## APB properties to observe
Picking up from Part 1 [[1](https://github.com/mjhborja/apply_stimuli_propagation_apb_part_1_uvm)], we are aware of the 3-signal handshake between IDLE, SETUP and ACCESS states. 
![diagram_004 1-stimuli_apb_p2_state_diagram_write](https://user-images.githubusercontent.com/50364461/213971542-e7c9972a-67a2-4d07-ac89-8a5e53920276.png)
\
The base line state machine diagram comes from [[5](https://documentation-service.arm.com/static/60d5b505677cf7536a55c245?token=)]. Only the annotations are Martin's orginal contribution.

To make the review systematic, it is highly recommended to tabulate the behavioral properties you will use to check the waveform.

|Code|Property name|Description|Reference [[5](https://documentation-service.arm.com/static/60d5b505677cf7536a55c245?token=)]|
|---|---|---|---|
|0|ERR_IDLE|Remains in the IDLE cycle, when PSEL stays low|Operating states from section 4.1|
|W1|ERR_WRITE_IDLE_SETUP|Coming from the IDLE cycle, when PSEL rises and PWRITE is high, PENABLE must be low in the SETUP cycle|Write transfers with no wait states from section 3.1.1|
|W2|ERR_WRITE_ACCESS|Coming from the SETUP cycle, PSEL stays high, PENABLE and PREADY are asserted, and all control signals and PWDATA are stable until the next rising edge of PCLK in the ACCESS cycle|Write transfers with no wait states from section 3.1.1|
|3|ERR_ACCESS_IDLE|Coming from the ACCESS cycle, PSEL and PENABLE are deasserted in the next sampling edge of PCLK in the IDLE cycle|Write transfers with no wait states from section 3.1.1|
|4|ERR_ACCESS_SETUP|Coming from the ACCESS cycle, PSEL stays high and PENABLE is deasserted in the next sampling edge of PCLK in the IDLE cycle|Write transfers with no wait states from section 3.1.1|
|W5|ERR_WRITE_WAIT_ACCESS|Coming from the SETUP cycle, PSEL stays high, PENABLE is asserted, PREADY is low, and all control signals and PWDATA are stable until the next rising edge of PCLK in the ACCESS cycle|Write transfers with wait states from section 3.1.2|
|W6|ERR_WRITE_WAIT_END_ACCESS|Coming from the ACCESS cycle, PSEL stays high, PENABLE and PREADY are asserted, and all control signals and PWDATA are stable until the next rising edge of PCLK in the ACCESS cycle|Write transfers with no wait states from section 3.1.2|
|R1|ERR_READ_IDLE_SETUP|Coming from the IDLE cycle, when PSEL rises and PWRITE is low, PENABLE must be low in the SETUP cycle|Read transfers with no wait states from section 3.3.1|
|R2|ERR_READ_ACCESS|Coming from the SETUP cycle, PSEL stays high, PENABLE and PREADY are asserted, all control signals are stable, and PRDATA is provided by the completer until the next rising edge of PCLK in the ACCESS cycle|Read transfers with no wait states from section 3.3.1|
|R5|ERR_READ_WAIT_ACCESS|oming from the SETUP cycle, PSEL stays high, PENABLE is asserted, PREADY is low, all control signals are stable until the next rising edge of PCLK in the ACCESS cycle|Write transfers with wait states from section 3.3.2|
|R6|ERR_READ_WAIT_END_ACCESS|Coming from the ACCESS cycle, PSEL stays high, PENABLE and PREADY are asserted, and all control signals and PRDATA are stable until the next rising edge of PCLK in the ACCESS cycle|Write transfers with no wait states from section 3.2.2|

Given the properties tabulated above, valid transitions for the 4 types of data transfers can be expanded from the operating states as follows:

### 1. Write with no wait cycles
\
![diagram_004 2-stimuli_apb_p2_write_no_wait](https://user-images.githubusercontent.com/50364461/213971586-a7f5fc56-8ba5-4eee-8607-ebd75337271d.png)
\
The base line timing diagram comes from [[5](https://documentation-service.arm.com/static/60d5b505677cf7536a55c245?token=)]. Only the annotations are Martin's orginal contribution.

### 2. Write with wait cycles
\
![diagram_004 3-stimuli_apb_p2_write_with_wait](https://user-images.githubusercontent.com/50364461/213971604-fe62eaee-9b07-4124-9a68-5c9cf2f752ce.png)
\
The base line timing diagram comes from [[5](https://documentation-service.arm.com/static/60d5b505677cf7536a55c245?token=)]. Only the annotations are Martin's orginal contribution.

### 3. Read with no wait cycles
\
![diagram_004 4-stimuli_apb_p2_read_no_wait](https://user-images.githubusercontent.com/50364461/213971621-0f06011d-ca9b-4193-a1db-b9b04cc7a7e3.png)
\
The base line timing diagram comes from [[5](https://documentation-service.arm.com/static/60d5b505677cf7536a55c245?token=)]. Only the annotations are Martin's orginal contribution.

### 4. Read with wait cycles
\
![diagram_004 5-stimuli_apb_p2_read_with_wait](https://user-images.githubusercontent.com/50364461/213971633-e1d2c5f6-640f-4a69-9b10-8f735beb776a.png)
\
The base line timing diagram comes from [[5](https://documentation-service.arm.com/static/60d5b505677cf7536a55c245?token=)]. Only the annotations are Martin's orginal contribution.

Note that each property stated above will be ignored if PRESETn is asserted. The combinations are tabulated below. To make transitions easier to understand, let's adopt |-> and |=> to stand for same cycle and next cycle.

|Code Transition|First Data Transfer|Next Data Transfer|
|---|---|---|
|0|idle|idle|
|0\|=>W1\|=>W2\|=>3|write no wait from idle|idle|
|0\|=>W1\|=>W2\|=>4|write no wait from idle|any non-idle data transfer|
|0\|=>W1\|=>n*W5\|=>W6\|=>3|write with n-wait cycles from idle|idle|
|0\|=>W1\|=>n*W5\|=>W6\|=>4|write with n-wait cycles from idle|any non-idle data transfer|
|0\|=>R1\|=>R2\|=>3|write no wait from idle|idle|
|0\|=>R1\|=>R2\|=>4|write no wait from idle|any non-idle data transfer|
|0\|=>R1\|=>n*R5\|=>R6\|=>3|read with n-wait cycles from idle|idle|
|0\|=>R1\|=>n*R5\|=>R6\|=>4|read with n-wait cycles from idle|any non-idle data transfer|
|4\|->W1\|=>W2\|=>3|write no wait from any non-idle data transfer|idle|
|4\|->W1\|=>W2\|=>4|write no wait from any non-idle data transfer|any non-idle data transfer|
|4\|->W1\|=>n*W5\|=>W6\|=>3|write with n-wait cycles from any non-idle data transfer|idle|
|4\|->W1\|=>n*W5\|=>W6\|=>4|write with n-wait cycles from any non-idle data transfer|any non-idle data transfer|
|4\|->R1\|=>R2\|=>3|write no wait from any non-idle data transfer|idle|
|4\|->R1\|=>R2\|=>4|write no wait from any non-idle data transfer|any non-idle data transfer|
|4\|->R1\|=>n*R5\|=>R6\|=>3|read with n-wait cycles from any non-idle data transfer|idle|
|4\|->R1\|=>n*R5\|=>R6\|=>4|read with n-wait cycles from any non-idle data transfer|any non-idle data transfer|

At this point, you already have a complete set of valid property combinations and transitions. And since the design complexity is quite low, with only 16, it is possible to do this manually for up to a few transactions. Here's the annotation of the waveform from Part 1 [[1](https://github.com/mjhborja/apply_stimuli_propagation_apb_part_1_uvm)].
![diagram_004 6-stimuli_apb_p2_waveform](https://user-images.githubusercontent.com/50364461/213971667-5c2108fa-2cab-42b3-a91e-be75d0c341b9.png)

And with the annotations in place, you may conclude that the VIP generates APB compliant stimuli within the scope of the transactions shown in the waveform.

Just bear in mind, that this exercise will let you familiarize with the process. And it is by no means intended that such checking be done manually as either the number of transactions or the design complexity scales up. But at some point, you will need to look things up on the waveform manually, as is the case when performing debug.

As you may observe the form of the tabulated properties of the interface protocol, you may explore the topic on protocol checking and SystemVerilog Assertions. But that is already outside the scope of this post and is worth another post.

## Key takeaways
__*uvm phase mapping to simulation outputs*__, __*behavioral property derivation*__, __*visual waveform checking*__

## References
[1] M. J. H. Borja, “Applying Stimuli Propagation to a Design - Part 1,” GitHub, Jan. 21, 2023. https://github.com/mjhborja/apply_stimuli_propagation_apb_part_1_uvm (accessed Jan. 23, 2023).
\
[2] “APB UVM With Scoreboard _,” EDA Playground. https://www.edaplayground.com/x/ueMH (accessed Jan. 19, 2023).
\
[3] M. J. H. Borja, Ed., “APB UVM test bench - Aldec Riviera Pro,” EDA Playground, Jan. 22, 2023. https://www.edaplayground.com/x/hf6p (accessed Jan. 23, 2023).
\
[4] Universal Verification Methodology (UVM) 1.2 Class Reference. California, U.S.A.: Accellera Systems Initiative Inc., 2014. Accessed: Jan. 21, 2023. [Online]. Available: https://www.accellera.org/images/downloads/standards/uvm/UVM_Class_Reference_Manual_1.2.pdf
\
[5] AMBA APB Protocol Specification. Cambridge, England: Arm Ltd., 2021. Accessed: Jan. 21, 2023. [Online]. Available: https://documentation-service.arm.com/static/60d5b505677cf7536a55c245?token=
\
[6] “UVM Phases,” ChipVerify. https://www.chipverify.com/uvm/uvm-phases (accessed Jan. 22, 2023).
\
[7] M. J. H. Borja, “UVM Hello World,” GitHub, Jan. 13, 2023. https://github.com/mjhborja/hello_world_uvm (accessed Jan. 19, 2023).
\
[8] C. Cummings and A. Salz, “SystemVerilog Event Regions, Race Avoidance & Guidelines,” Synopsys User Group (SNUG), Sep. 2006. Accessed: Jan. 22, 2023. [Online]. Available: http://www.sunburst-design.com/papers/CummingsSNUG2006Boston_SystemVerilog_Events.pdf

## Note to reader
The review content, including references to the original source code, analysis, diagrams otherwise stated and overall presentation of the topic, is Martin's original contribution. The base line source code and diagrams for this post are not Martin's work.
