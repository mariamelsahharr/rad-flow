Create Your First RAD-Sim Module
=================

Overview
---------------------------------

In this page, we will walkthrough the steps to create a minimal RAD-sim transmitter module that sends a single integer to an reciever on NoC, to help you create/adapt your own designs.

Here's a one line summary for each module we will create:

1. ``sender``: specifies how to send the integer via AXI master interface to the NoC.

2. ``receiver``: specifies how to receive the integer from the NoC via AXI slave interface

3. ``transmitter_top``: connects the sender and receiver modules together, and instantiates the NoC according to specs.

4. ``transmitter_system``: usually connects top level with testbench, in this case we don't have a seperate testbench, so only top level is instantiated.

We will also create the following files for configuring the design:

1. ``transmitter.clks``: specifies the clock and reset signals for the design.

2. ``transmitter.place``: specifies the placement of the modules on the NoC

3. ``config.yml``: specifies the NoC parameters, RAD sepecific parameters and their placement on RAD clusters.

4. ``CMakeLists.txt``: A filelist that specifies the source and header files for the design.

Before we start, let's create our working directory first:

    .. code-block:: bash

        $ mkdir <rad_flow_root_dir>/rad-sim/example-designs/transmitter/modules 
        

Sender Module
-----------------

The sender module sends an integer to an recevier via the NoC, it should inherit from the ``RADSimModule`` class, instantiate an AXI master interface, and override the ``RegisterModuleInfo`` method to register the AXI master port. We should also have methods for its sequential and combinational logic.

Let's create a header file accordingly under ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/modules/sender.hpp``

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

        class sender : public RADSimModule {
        public:
            RADSimDesignContext* radsim_design;
            sc_in<bool> rst;

            // Interface to the NoC
            axis_master_port axis_sender_interface;
            
            sender(const sc_module_name &name, RADSimDesignContext* radsim_design);
            ~sender();
            
            void Assign(); // Combinational logic process
            void Tick();   // Sequential logic process
            SC_HAS_PROCESS(sender);
            void RegisterModuleInfo();
        };

Next, we implement these methods in ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/modules/sender.cpp``, let's walk through each method:


Starting with the constructor, destructor and RegisterModuleInfo, the important thing here is to mark our combinitional and sequential methods(``Assign`` and ``Tick``) with the appropriate SystemC macros, and to register the AXI master port with the design context. Note that we must have RegisterModuleInfo seperately since it is required by the flow, the destructor is also necessary since the parent class defined it as virtual method.
   
    .. code-block:: c++

        sender::sender(const sc_module_name &name, RADSimDesignContext* radsim_design)
              : RADSimModule(name, radsim_design) {
            this->radsim_design = radsim_design;
            SC_METHOD(Assign);
            sensitive << rst;

            SC_CTHREAD(Tick, clk.pos());
            reset_signal_is(rst, true); // Reset is active high
    
            RegisterModuleInfo();
        }

        sender::~sender() {
            // Destructor does not need to do anything for this module
        }

        void sender::RegisterModuleInfo() {
            std::string port_name = module_name + ".axis_sender_interface";
            RegisterAxisMasterPort(port_name, &axis_sender_interface, 128, 0);
        }

Next, we implement the combinational logic in the ``Assign`` method, in this example it only needs to deal with reset:
    
    .. code-block:: c++

        void sender::Assign() {
            if (rst) {
                axis_sender_interface.tvalid.write(false);  
            }
        }

Finally, we implement the sequential logic in the ``Tick`` method, which is where we send the integer to the NoC.

A couple things to note here:

- This is a very simplified implementation where we just send a single number, it's meant only to demonstrate the flow.

- For AXIS interface, we need to concatinate the remote node ID, local node ID and RAD ID into a single destination ID, which is then written to the ``tdest`` port of the AXIS interface.

- We also need to supply the src_addr to the ``tuser`` port, which in RADSim is used for the destination address of the AXIS packet.

    .. code-block:: c++

        void sender::Tick() {
            wait();
            int start_cycle = GetSimulationCycle(radsim_config.GetDoubleKnobShared("sim_driver_period"));

            while (true) {        
                std::string src_port_name = module_name + ".axis_sender_interface";
                std::string dst_port_name = "receiver_inst.axis_receiver_interface";
                uint64_t dst_addr = radsim_design->GetPortDestinationID(dst_port_name);
                uint64_t src_addr = radsim_design->GetPortDestinationID(src_port_name);
                sc_bv<AXIS_DESTW> dest_id_concat;
                DEST_REMOTE_NODE(dest_id_concat) = 0; // staying on same RAD
                DEST_LOCAL_NODE(dest_id_concat) = dst_addr;
                DEST_RAD(dest_id_concat) = radsim_design->rad_id;
                unsigned int placeholderData = 1;
                axis_sender_interface.tdest.write(dest_id_concat);
                axis_sender_interface.tid.write(0);
                axis_sender_interface.tstrb.write(0);
                axis_sender_interface.tkeep.write(0);
                axis_sender_interface.tuser.write(src_addr);
                axis_sender_interface.tlast.write(1);
                axis_sender_interface.tdata.write(placeholderData);
                axis_sender_interface.tvalid.write(true);
                wait();

                int end_cycle = GetSimulationCycle(radsim_config.GetDoubleKnobShared("sim_driver_period"));
                
                if(end_cycle - start_cycle > 50){            
                    break;
                }
            }
            this->radsim_design->set_rad_done();
        }

Reciever Module
-----------------

The reciever pretty much mirrors the same template, except it uses an AXI slave interface to receive the integer from the NoC.

Let's create a header file accordingly under ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/modules/receiver.hpp``

    .. code-block:: c++

        #pragma once

        #include <axis_interface.hpp>
        #include <design_context.hpp>        
        #include <radsim_defines.hpp>
        #include <radsim_module.hpp>
        #include <string>
        #include <systemc.h>        
        #include <radsim_utils.hpp> 
        #include <vector>       

        class receiver: public RADSimModule {
        public:
            RADSimDesignContext* radsim_design;
            sc_in<bool> rst;    
            // Interface to the NoC
            axis_slave_port axis_receiver_interface;

            receiver(const sc_module_name &name, RADSimDesignContext* radsim_design);
            ~receiver();
            void Assign(); // Combinational logic process
            void Tick();   // Sequential logic process
            SC_HAS_PROCESS(receiver);
            void RegisterModuleInfo();
        };

Then we implement these methods in ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/modules/receiver.cpp``

    .. code-block:: c++

        #include <receiver.hpp>

        receiver::receiver(const sc_module_name &name, RADSimDesignContext* radsim_design)
            : RADSimModule(name, radsim_design) {
            this->radsim_design = radsim_design;

            SC_METHOD(Assign);
            sensitive << rst;

            SC_CTHREAD(Tick, clk.pos());
            reset_signal_is(rst, true); // Reset is active high

            RegisterModuleInfo();
        }

        receiver::~receiver() {
            // Destructor does not need to do anything for this module
        }

        void receiver::Tick() {
            wait();
            while (true) {
                if (axis_receiver_interface.tvalid.read() && axis_receiver_interface.tready.read()) {
                    sc_bv<128> tdata = axis_receiver_interface.tdata.read();
                    bool tlast = axis_receiver_interface.tlast.read();
                    std::cout << "Received Data: " << tdata.to_string() << ", Last: " << tlast << std::endl;
                }
                wait();
            }
        }

        void receiver::Assign() {
            if (rst) {
                axis_receiver_interface.tready.write(false);
            } else {
                // Always ready to receive data
                axis_receiver_interface.tready.write(true);
            }
        }

        void receiver::RegisterModuleInfo() {
            std::string port_name = module_name + ".axis_receiver_interface";
            RegisterAxisSlavePort(port_name, &axis_receiver_interface, 128, 0);
        }

Transmitter Top
-----------------
The transmitter top module connects the sender and receiver modules together, and instantiates the NoC according to specs. It should inherit from ``RADSimDesignTop``

header file at ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/transmitter_top.hpp``

    .. code-block:: c++

        #pragma once

        #include <radsim_config.hpp>
        #include <sender.hpp>
        #include <receiver.hpp>
        #include <systemc.h>
        #include <vector>
        #include <design_top.hpp>
        #include <axis_interface.hpp>

        class transmitter_top : public RADSimDesignTop {
        private:
            sender *sender_inst;
            receiver *receiver_inst;
            RADSimDesignContext* radsim_design;
        public:
            sc_in<bool> rst;
            transmitter_top(const sc_module_name &name, RADSimDesignContext* radsim_design);
            ~transmitter_top();
        };

implementation at ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/transmitter_top.cpp``

    .. code-block:: c++

        #include <transmitter_top.hpp>

        transmitter_top::transmitter_top(const sc_module_name &name, RADSimDesignContext* radsim_design)
            : RADSimDesignTop(radsim_design) {

            this->radsim_design = radsim_design;

            std::string module_name_str;
            char module_name[25];

            module_name_str = "sender_inst";
            std::strcpy(module_name, module_name_str.c_str());

            sender_inst = new sender(module_name, radsim_design);
            sender_inst->rst(rst);

            module_name_str = "receiver_inst";
            std::strcpy(module_name, module_name_str.c_str());

            receiver_inst = new receiver(module_name, radsim_design);
            receiver_inst->rst(rst);

            this->connectPortalReset(&rst);
            radsim_design->BuildDesignContext("transmitter.place", "transmitter.clks");
            radsim_design->CreateSystemNoCs(rst);
            radsim_design->ConnectModulesToNoC();
        }

        transmitter_top::~transmitter_top() {
            delete sender_inst;
            delete receiver_inst;
        }

Transmitter System
-----------------
This is the last and topmost module we have, usually this connects the driver, scoreboard etc with the top level design, in this case we don't have a separate testbench, so only top level is instantiated. 

header file at ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/transmitter_system.hpp``

    .. code-block:: c++

        #pragma once

        #include <transmitter_top.hpp>
        #include <chrono>
        #include <vector>
        #include <design_system.hpp>

        class transmitter_system : public RADSimDesignSystem {
            public:
            sc_signal<bool> rst_sig;
            sc_clock *sysclk;
            transmitter_top *dut_inst;

            transmitter_system(const sc_module_name &name, sc_clock *driver_clk_sig, RADSimDesignContext* radsim_design);
            ~transmitter_system();
        };

implementation at ``<rad_flow_root_dir>/rad-sim/example-designs/transmitter/transmitter_system.cpp``

    .. code-block:: c++
        
        #include <transmitter_system.hpp>

        transmitter_system::transmitter_system(const sc_module_name &name, sc_clock *driver_clk_sig, RADSimDesignContext* radsim_design) 
            : sc_module(name) {

            // Instantiate design top-level
            dut_inst = new transmitter_top("dut", radsim_design);
            dut_inst->rst(rst_sig);
            this->design_dut_inst = dut_inst;
        }

            transmitter_system::~transmitter_system() {
            delete dut_inst;
            delete sysclk;
        }

Configurations
---------------------------------
There are a couple configuration files needed for each design, the complete structure of each is covered in :doc:`rad-sim-code-structure`, only important settings will be explained here.

**transmitter.place:**

Here we set the sender to position 0 of NOC 0, and the reciever to position 3 of the same NOC

    .. code-block:: text
        
        sender_inst 0 0 axis
        receiver_inst 0 3 axis

**transmitter.clks:**

Here we set sender and reciever to the clk frequency(5 ns period as defined in config.yml)

    .. code-block:: text

        sender_inst 0 0
        receiver_inst 0 0

**config.yml:**

Various settings for NOC and the design, here we copied most of the settings and only chaned design related ones

    .. code-block:: yaml

        config rad1:
            design:
                name: 'transmitter'
                noc_placement: ['transmitter.place']
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
            transmitter_system.cpp
            transmitter_top.cpp
        )

        set(hdrfiles
            modules/sender.hpp
            modules/receiver.hpp
            transmitter_system.hpp
            transmitter_top.hpp
        )

        add_compile_options(-Wall -Wextra -pedantic)

        add_library(transmitter STATIC ${srcfiles} ${hdrfiles})
        target_link_libraries(transmitter PUBLIC SystemC::systemc booksim noc)

Running the Design
---------------------------------

Run these on the command line:

.. code-block:: bash

    $ cd <rad_flow_root_dir>/rad-sim/
    $ python config.py transmitter
    $ cd build
    $ cmake ..
    $ make