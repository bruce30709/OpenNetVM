#                    openNetVM
#      https://github.com/sdnfv/openNetVM
#
# BSD LICENSE
#
# Copyright(c)
#          2015-2017 George Washington University
#          2015-2017 University of California Riverside
#          2016-2017 Hewlett Packard Enterprise Development LP
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
# Redistributions of source code must retain the above copyright
# notice, this list of conditions and the following disclaimer.
# Redistributions in binary form must reproduce the above copyright
# notice, this list of conditions and the following disclaimer in
# the documentation and/or other materials provided with the
# distribution.
# The name of the author may not be used to endorse or promote
# products derived from this software without specific prior
# written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
# OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
# DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

ifeq ($(RTE_SDK),)
$(error "Please define RTE_SDK environment variable")
endif

# To add new examples, append the directory name to this variable
examples = bridge basic_monitor simple_forward speed_tester flow_table test_flow_dir aes_encrypt aes_decrypt flow_tracker load_balancer arp_response nf_router scaling_example load_generator payload_scan firewall simple_fwd_tb l2fwd busy_forward busy_firewall SFC_measurment

ifeq ($(NDPI_HOME),)
$(warning "Skipping ndpi_stats NF as NDPI_HOME is not set")
else
examples +=ndpi_stats
endif

clean_examples=$(addprefix clean_,$(examples))

.PHONY: $(examples) $(clean_examples)

all : $(examples)
clean: $(clean_examples)

$(examples):
	cd $@ && $(MAKE)

$(clean_examples):
	cd $(patsubst clean_%,%,$@) && $(MAKE) clean
