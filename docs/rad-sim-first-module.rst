Create Your First RAD-Sim Module
=================

Overview
---------------------------------

In this page, we will walkthrough the steps to create a minimal RAD-sim producer_consumer module that sends a series of integers to a reciever over the NoC, to help you create/adapt your own designs.

Here's a one line summary for each module we will create:

1. ``sender``: specifies how to send the integers via AXI master interface to the NoC.

2. ``receiver``: specifies how to receive the integer from the NoC via AXI slave interface

3. ``producer_consumer_top``: connects the sender and receiver modules together, and instantiates the NoC according to specs.

4. ``producer_consumer_system``: usually connects top level with testbench, in this case we don't have a seperate testbench, so only top level is instantiated.

We will also create the following files for configuring the design:

1. ``producer_consumer.clks``: specifies the clock and reset signals for the design.

2. ``producer_consumer.place``: specifies the placement of the modules on the NoC

3. ``config.yml``: specifies the NoC parameters, RAD sepecific parameters and their placement on RAD clusters.

4. ``CMakeLists.txt``: A filelist that specifies the source and header files for the design.

Before we start, let's create our working directory first:

    .. code-block:: bash

        $ mkdir <rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/modules 
        

Sender Module
-----------------

The sender module sends an integer to a recevier via the NoC, it should inherit from the ``RADSimModule`` class, instantiate an AXI master interface, and override the ``RegisterModuleInfo`` method to register the AXI master port. We should also have methods for its sequential and combinational logic.

Let's create a header file accordingly under ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/modules/sender.hpp``

    .. code-block:: c++

        #pragma once

        #include <axis_interface.hpp>
        #include <design_context.hpp>
        #include <radsim_defines.hpp>
        #include <radsim_module.hpp>
        #include <string>
        #include <systemc.h>
        #include <vector>
        #include <radsim_utils.hpp>
        #include <fstream>
        #include <iostream>

        class sender : public RADSimModule {
        private:
            std::ofstream outfile;
            //the total amount of integers to send
            int amount_to_send; 
            //wait this many cycles after sending the last integer before signaling done
            int wait_cycles;        
        public:
            //The context contain utilities for generating design, NOC, AXI adaptor clks, connecting AXI interfaces to the NoC, etc.
            RADSimDesignContext* radsim_design;
            sc_in<bool> rst;    

            // Interface to the NoC
            axis_master_port axis_sender_interface;
            
            sender(const sc_module_name &name, RADSimDesignContext* radsim_design, int amount_to_send = 10, int wait_cycles = 50);
            ~sender();
            
            void Assign(); // Combinational logic process
            void Tick();   // Sequential logic process
            SC_HAS_PROCESS(sender);
            void RegisterModuleInfo();
        };

Next, we implement these methods in ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/modules/sender.cpp``. Let's walk through each method:


Starting with the constructor, destructor and RegisterModuleInfo, the important thing here is to mark our combinitional and sequential methods(``Assign`` and ``Tick``) with the appropriate SystemC macros, and to register the AXI master port with the design context. Note that we must have RegisterModuleInfo seperately since it is required by the flow, the destructor is also necessary since the parent class defined it as virtual method.
   
    .. code-block:: c++

        sender::sender(const sc_module_name &name, RADSimDesignContext* radsim_design, int amount_to_send, int wait_cycles)
            : RADSimModule(name, radsim_design) {
            this->radsim_design = radsim_design;
            this->amount_to_send = amount_to_send;
            this->wait_cycles = wait_cycles;
            SC_METHOD(Assign);
            //Sets sensitivity for combinational logic in Assign method, similiar to always_comb block in System Verilog
            sensitive << rst;

            SC_CTHREAD(Tick, clk.pos());
            reset_signal_is(rst, true); // Reset is active high
            
            RegisterModuleInfo();
            //starts a file to record the output, later we can use this to verify on the receiver side
            outfile.open("sender_output.txt");
        }

        sender::~sender() {
            // Destructor does not need to do anything for this module
            if (outfile.is_open()) {
                outfile.close();
            }
        }

        void sender::RegisterModuleInfo() {
            std::string port_name = module_name + ".axis_sender_interface";
            RegisterAxisMasterPort(port_name, &axis_sender_interface, 128, 0);
        }

Next, we can put any combinational logic in the ``Assign`` method, in this example it doesn't need any, this is purely to show where to do it.
    
    .. code-block:: c++

        void sender::Assign() {
            //cmbination logical here
        }

Finally, we implement the sequential logic in the ``Tick`` method, which is where we send the integer to the NoC.

A couple things to note here:

- This is a very simplified implementation where we just send a sequence of numbers, it's meant only to demonstrate the flow.

- For AXIS interface, we need to concatinate the remote node ID, local node ID and RAD ID into a single destination ID, which is then written to the ``tdest`` port of the AXIS interface.

- We also need to supply the src_addr to the ``tuser`` port, which in RADSim is used for tracking the AXIS packet.

    .. code-block:: c++

        void sender::Tick() {
            if(rst) {
                axis_sender_interface.tvalid.write(false);  
            }
            wait();

            int data_left = amount_to_send;
            unsigned int placeholderData = 0;

            while (data_left > 0) {       
                //collect the src and destination data, then write to the AXIS interface
                std::string src_port_name = module_name + ".axis_sender_interface";
                std::string dst_port_name = "receiver_inst.axis_receiver_interface";
                uint64_t dst_addr = radsim_design->GetPortDestinationID(dst_port_name);
                uint64_t src_addr = radsim_design->GetPortDestinationID(src_port_name);
                sc_bv<AXIS_DESTW> dest_id_concat;
                DEST_REMOTE_NODE(dest_id_concat) = 0; // staying on same RAD
                DEST_LOCAL_NODE(dest_id_concat) = dst_addr;
                DEST_RAD(dest_id_concat) = radsim_design->rad_id;
                placeholderData += 1;
                axis_sender_interface.tdest.write(dest_id_concat);
                axis_sender_interface.tid.write(0);
                axis_sender_interface.tstrb.write(0);
                axis_sender_interface.tkeep.write(0);
                axis_sender_interface.tuser.write(src_addr);
                //prints the output at each cycle
                std::cout << "@cycle " << GetSimulationCycle(radsim_config.GetDoubleKnobShared("sim_driver_period")) << " Sending " << placeholderData << " to destination: " << dst_addr << std::endl;
                if(data_left > 1) {
                    axis_sender_interface.tlast.write(0); // Not the last data
                } else {
                    axis_sender_interface.tlast.write(1); // Last data
                }        
                axis_sender_interface.tdata.write(placeholderData);
                axis_sender_interface.tvalid.write(true);
                wait();
                //records the data sent
                outfile << placeholderData << std::endl;

                data_left--; 

            }
            axis_sender_interface.tvalid.write(false); // Signal that no more data will be sent
            int wait_start_cycle = GetSimulationCycle(radsim_config.GetDoubleKnobShared("sim_driver_period"));
            int curr_cycle = wait_start_cycle;

            while(curr_cycle < wait_start_cycle + wait_cycles) {
                curr_cycle = GetSimulationCycle(radsim_config.GetDoubleKnobShared("sim_driver_period"));
                wait();
            }
            //signals the end of simulation for this module
            this->radsim_design->set_rad_done();
        }

Reciever Module
-----------------

The reciever pretty much mirrors the same template, except it uses an AXI slave interface to receive the integer from the NoC.

Let's create a header file accordingly under ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/modules/receiver.hpp``

    .. code-block:: c++

        #pragma once

        #include <axis_interface.hpp>
        #include <design_context.hpp>
        #include <queue>
        #include <radsim_defines.hpp>
        #include <radsim_module.hpp>
        #include <string>
        #include <systemc.h>
        #include <vector>
        #include <radsim_utils.hpp>
        #include <tuple>
        #include <fstream>
        #include <iostream>

        class receiver: public RADSimModule {
        private:
            std::ifstream refFile;
            bool errorFlag;
        public:
            RADSimDesignContext* radsim_design;
            sc_in<bool> rst;    
            // Interface to the NoC
            axis_slave_port axis_receiver_interface;

            receiver(const sc_module_name &name, RADSimDesignContext* radsim_design);
            ~receiver();    
            void Tick();   // Sequential logic process
            SC_HAS_PROCESS(receiver);
            void RegisterModuleInfo();
        };

Then we implement these methods in ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/modules/receiver.cpp``

    .. code-block:: c++

        #include <receiver.hpp>

        receiver::receiver(const sc_module_name &name, RADSimDesignContext* radsim_design)
            : RADSimModule(name, radsim_design) {
            this->radsim_design = radsim_design;
        
            sensitive << rst;

            SC_CTHREAD(Tick, clk.pos());
            reset_signal_is(rst, true); // Reset is active high

            RegisterModuleInfo();
            refFile.open("sender_output.txt");
        }

        receiver::~receiver() {
            //check if any data is left in the reference file
            std::string line;
            if(std::getline(refFile, line)) {
                std::cout << "Some data have not arrived!" << std::endl;
                errorFlag = true;
            }
            if (errorFlag) {
                std::cerr << "Error occurred during data reception!" << std::endl;
            } else {
                std::cout << "Data reception completed successfully!" << std::endl;
            }
            refFile.close();
        }

        void receiver::Tick() {
            if (rst) {
                axis_receiver_interface.tready.write(false);  
            } else {
                axis_receiver_interface.tready.write(true); // Ready to receive data
            }
            wait();
            errorFlag = false;
            while (true) {
                if (axis_receiver_interface.tvalid.read() && axis_receiver_interface.tready.read()) {
                    unsigned int tdata = axis_receiver_interface.tdata.read().range(31, 0).to_uint();;
                    bool tlast = axis_receiver_interface.tlast.read();
                    std::cout << "Received Data: " << std::to_string(tdata) << ", Last: " << tlast << std::endl;
                    //compare with reference file
                    if (refFile.is_open()) {
                        std::string line;
                        if (std::getline(refFile, line)) {
                            if (line != std::to_string(tdata)) {
                                std::cout << "Data mismatch! Expected: " << line << ", Received: " << std::to_string(tdata) << std::endl;
                                errorFlag = true;
                            }
                        } else {
                            std::cout << "No more lines in reference file." << std::endl;
                            errorFlag = true;
                        }
                    } else {
                        std::cerr << "Reference file not open!" << std::endl;
                        errorFlag = true;
                    }
                    
                }
                wait();
            }
        }


        void receiver::RegisterModuleInfo() {
            std::string port_name = module_name + ".axis_receiver_interface";
            RegisterAxisSlavePort(port_name, &axis_receiver_interface, 128, 0);
        }

Producer_Consumer Top
-----------------
The producer_consumer top module connects the sender and receiver modules together, and instantiates the NoC according to specs. It should inherit from ``RADSimDesignTop``

Let's define header file at ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/producer_consumer_top.hpp``

    .. code-block:: c++

        #pragma once

        #include <radsim_config.hpp>
        #include <sender.hpp>
        #include <receiver.hpp>
        #include <systemc.h>
        #include <vector>
        #include <design_top.hpp>
        #include <axis_interface.hpp>

        class producer_consumer_top : public RADSimDesignTop {
        private:
            sender *sender_inst;
            receiver *receiver_inst;
            RADSimDesignContext* radsim_design;
        public:
            sc_in<bool> rst;
            producer_consumer_top(const sc_module_name &name, RADSimDesignContext* radsim_design);
            ~producer_consumer_top();
        };

Then implement at at ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/producer_consumer_top.cpp``, notice that we dump the NoC througput stats at the destructor

    .. code-block:: c++

        #include <producer_consumer_top.hpp>

        producer_consumer_top::producer_consumer_top(const sc_module_name &name, RADSimDesignContext* radsim_design)
            : RADSimDesignTop(radsim_design) {

            this->radsim_design = radsim_design;

            std::string module_name_str;
            char module_name[25];

            // Create the sender and receiver instances
            module_name_str = "sender_inst";
            std::strcpy(module_name, module_name_str.c_str());

            sender_inst = new sender(module_name, radsim_design);
            sender_inst->rst(rst);

            module_name_str = "receiver_inst";
            std::strcpy(module_name, module_name_str.c_str());

            receiver_inst = new receiver(module_name, radsim_design);
            receiver_inst->rst(rst);

            // Connect the sender and receiver to the design context
            this->connectPortalReset(&rst);
            radsim_design->BuildDesignContext("producer_consumer.place", "producer_consumer.clks");
            radsim_design->CreateSystemNoCs(rst);
            radsim_design->ConnectModulesToNoC();
        }

        //Dumps Noc Throughput stats
        producer_consumer_top::~producer_consumer_top() {
            end_cycle = GetSimulationCycle(radsim_config.GetDoubleKnobShared("sim_driver_period"));
            NoCTransactionTelemetry::DumpStatsToFile("stats.csv");
            NoCFlitTelemetry::DumpNoCFlitTracesToFile("flit_traces.csv");

            std::vector<double> aggregate_bandwidths = NoCTransactionTelemetry::DumpTrafficFlows("traffic_flows", 
                end_cycle - start_cycle, radsim_design->GetNodeModuleNames(), radsim_design->rad_id);
            std::cout << "Aggregate NoC BW = " << aggregate_bandwidths[0] / 1000000000 << " Gbps" << std::endl;

            delete sender_inst;
            delete receiver_inst;
        }

Producer_Consumer System
-----------------
This is the last and topmost module we have, usually this connects the driver and other testbench components with the top level design, in this case we don't have a separate testbench, so only top level is instantiated. 

Let's define the header file at ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/producer_consumer_system.hpp``

    .. code-block:: c++

        #pragma once

        #include <producer_consumer_top.hpp>
        #include <chrono>
        #include <vector>
        #include <design_system.hpp>

        class producer_consumer_system : public RADSimDesignSystem {
            public:
            sc_signal<bool> rst_sig;
            sc_clock *sysclk;
            producer_consumer_top *dut_inst;

            producer_consumer_system(const sc_module_name &name, sc_clock *driver_clk_sig, RADSimDesignContext* radsim_design);
            ~producer_consumer_system();
        };

Then implement at ``<rad_flow_root_dir>/rad-sim/example-designs/producer_consumer/producer_consumer_system.cpp``

    .. code-block:: c++
        
        #include <producer_consumer_system.hpp>

        producer_consumer_system::producer_consumer_system(const sc_module_name &name, sc_clock *driver_clk_sig, RADSimDesignContext* radsim_design) 
            : sc_module(name) {

            // Instantiate design top-level
            dut_inst = new producer_consumer_top("dut", radsim_design);
            dut_inst->rst(rst_sig);
            this->design_dut_inst = dut_inst;
        }

            producer_consumer_system::~producer_consumer_system() {
            delete dut_inst;
            delete sysclk;
        }

Configurations
---------------------------------
There are a couple configuration files needed for each design, the complete structure of each is covered in :doc:`rad-sim-code-structure`, only important settings will be explained here.

**producer_consumer.place:**

Here we set the sender to position 0 of NOC 0, and the reciever to position 3 of the same NOC, axis means we're using AXI Streaming interface, we can also use aximm(AXI Memory Mapped) interface if needed.

    .. code-block:: text
        
        sender_inst 0 0 axis
        receiver_inst 0 3 axis

**producer_consumer.clks:**

Here we set sender and reciever to the clk frequency(5 ns period as defined in config.yml)

    .. code-block:: text

        sender_inst 0 0
        receiver_inst 0 0

**config.yml:**

This YAML file configures all the RAD-Sim parameters for the simulation of the application design under 4 main tags: 
``noc``, ``noc_adapters``, ``config <configname>``, and ``cluster``. The ``noc`` and ``noc_adapters`` parameters are shared across all RADs. 
There may be multiple ``config <configname>`` sections, each describing a RAD configuration that can be applied to a single or multiple devices in the cluster.
The ``cluster`` tag describes the cluster of RADs, including the number of RADs and their configurations. 

This file should be located in the same directory as the ``config.py`` script. For a new design, you should copy
the ``config.yml`` file from one of the provided example design directories and make modifications for your use case. 

Note that the parameters within a ``config <configname>`` subsection can be applied to a single RAD or shared among multiple RADs.
An example configuration file is shown below, followed by an explanation for each configuration paramete

    .. code-block:: yaml

        config rad1:
            design:
                name: 'producer_consumer'
                noc_placement: ['producer_consumer.place']
                clk_periods: [5.0]

        noc:
            type: ['2d']
            num_nocs: 1
            clk_period: [1.0]
            payload_width: [166]
            topology: ['mesh']
            dim_x: [4]
            dim_y: [4] 
            routing_func: ['dim_order']
            vcs: [5]
            vc_buffer_size: [8]
            output_buffer_size: [8]
            num_packet_types: [5]
            router_uarch: ['iq']
            vc_allocator: ['islip']
            sw_allocator:  ['islip']
            credit_delay: [1]
            routing_delay: [1]
            vc_alloc_delay: [1]
            sw_alloc_delay: [1]

        noc_adapters:
            clk_period: [1.25]
            fifo_size: [16]
            obuff_size: [2]
            in_arbiter: ['fixed_rr']
            out_arbiter: ['priority_rr']
            vc_mapping: ['direct']


        cluster:
            sim_driver_period: 5.0
            telemetry_log_verbosity: 2
            telemetry_traces: []
            num_rads: 1
            cluster_configs: ['rad1']

**CmakeLists.txt:**

This is a filelist that specifies the source and header files for the design, as well as other includes

    .. code-block:: cmake

        cmake_minimum_required(VERSION 3.16)
        find_package(SystemCLanguage CONFIG REQUIRED)

        include_directories(
            ./
            modules
            ../../sim
            ../../sim/noc
            ../../sim/noc/booksim
            ../../sim/noc/booksim/networks
            ../../sim/noc/booksim/routers
        )

        set(srcfiles
            modules/sender.cpp
            modules/receiver.cpp
            producer_consumer_system.cpp
            producer_consumer_top.cpp
        )

        set(hdrfiles
            modules/sender.hpp
            modules/receiver.hpp
            producer_consumer_system.hpp
            producer_consumer_top.hpp
        )

        add_compile_options(-Wall -Wextra -pedantic)

        add_library(producer_consumer STATIC ${srcfiles} ${hdrfiles})
        target_link_libraries(producer_consumer PUBLIC SystemC::systemc booksim noc)

Running the Design
---------------------------------

Run the following commands in the terminal:

.. code-block:: bash

    $ cd <rad_flow_root_dir>/rad-sim/
    $ python config.py producer_consumer
    $ cd build
    $ cmake ..
    $ make

Expected output:

.. code-block:: text

    placement_filepath: /home/kerry/rad-flow/rad-sim/example-designs/producer_consumer/producer_consumer.place
    @cycle 1 Sending 1 to destination: 1
    @cycle 2 Sending 2 to destination: 1
    @cycle 3 Sending 3 to destination: 1
    @cycle 4 Sending 4 to destination: 1
    @cycle 5 Sending 5 to destination: 1
    @cycle 6 Sending 6 to destination: 1
    @cycle 7 Sending 7 to destination: 1
    Received Data: 1, Last: 0
    @cycle 8 Sending 8 to destination: 1
    Received Data: 2, Last: 0
    @cycle 9 Sending 9 to destination: 1
    Received Data: 3, Last: 0
    @cycle 10 Sending 10 to destination: 1
    Received Data: 4, Last: 0
    Received Data: 5, Last: 0
    Received Data: 6, Last: 0
    Received Data: 7, Last: 0
    Received Data: 8, Last: 0
    Received Data: 9, Last: 0
    Received Data: 10, Last: 1

    Info: /OSCI/SystemC: Simulation stopped by user.
    Simulation Cycles from main.cpp = 63
    Aggregate NoC BW = 4.06349 Gbps
    Data reception completed successfully!