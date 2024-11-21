1.	For each member in your team, provide 1 paragraph detailing what parts of the lab that member implemented / researched. (You may skip this question if you are doing the lab by yourself).
Ans. This Assignment was entirely done by me 

2.	Describe in detail the steps you used to complete the assignment. Consider your reader to be someone skilled in software development but otherwise unfamiliar with the assignment. Good answers to this question will be recipes that someone can follow to reproduce your development steps.

Ans.
I will explain the procedure I followed in Steps:

Step 1: GIT
•	Fork the repository into your git 
•	Then clone the repository into our virtual machine(VM) 
git clone https://github.com/PratikKM2002/linux.git
cd linux
              Step 2: Create Kernel configuration
•	After the git part is done we now focus on building our custom kernel
•	Update the Package List: sudo apt-get update
•	Install Kernel Build Dependencies: sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev
•	Install lz4 (Compression Library): sudo apt-get install lz4
•	Run Kernel Configuration: make config
•	The provided command generates a new self-signed X.509 certificate and private key using OpenSSL. This is often used in the context of enabling Secure Boot for a Linux kernel:   openssl req -new -x509 -newkey rsa:2048 -sha256 -nodes -days 365 -keyout debian/certs/debian-uefi-certs.key -out debian/certs/debian-uefi-certs.pem -subj "/CN= Debian Secure Boot Certificate
             Step 3: Building the custom kernel
•	Build the Kernel: 
                    make -j$(nproc)
•	Install Kernel Modules:
       sudo make modules_install

•	Install the Kernel:
    sudo make install

•	Reboot the System:
sudo reboot

•	After reboot, the system usually boots into the newest kernel by default,we can check by using

uname -r
  
 Step 4: Make changes to the vmx.c to add counters
•	Navigate to linux/arch/x86/kvm/vmx
•	Use Vim or Nano to edit the code
•	Edit vmx.c file to add counters to the code
•	Find the function which is handling the exits
•	In our case the function name is __vmx_handle_exit function
•	Add #include<linux/printk.h> to beginning of vmx.c
•	Modify the function to add the following to it

static unsigned long long exit_type_counters[256] = {0}; // Array for per-exit-type counters
static unsigned long long total_exit_count = 0;         // Global total exit counter


static const char *kvm_exit_reason_to_string(int reason) {
    switch (reason) {
    case 0: return "EXCEPTION_NMI";
    case 1: return "EXTERNAL_INTERRUPT";
    case 2: return "TRIPLE_FAULT";
    case 9: return "CPUID";
    case 28: return "HLT";
    case 30: return "MSR_WRITE";
    case 48: return "EPT_MISCONFIG";
    default: return "UNKNOWN_EXIT";
    }
}





static int __vmx_handle_exit(struct kvm_vcpu *vcpu, fastpath_t exit_fastpath) {
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    union vmx_exit_reason exit_reason = vmx->exit_reason;
    u32 vectoring_info = vmx->idt_vectoring_info;
    u16 exit_handler_index;

    /* Add the counters at the start of the function */
    int exit_reason_type = exit_reason.basic; // Get the exit reason type number
    exit_type_counters[exit_reason_type]++;
    total_exit_count++;

    /* Print every 10,000 total exits */
    if (total_exit_count % 10000 == 0) {
        int i;
        for (i = 0; i < 256; i++) {
            if (exit_type_counters[i] > 0) {
                printk(KERN_INFO "KVM Exit: %d (%s) occurred %llu times\n",
                       i, kvm_exit_reason_to_string(i), exit_type_counters[i]);
            }
        }
    }

    /* Continue with the rest of the normal exit handling logic */
    ...
}


•	Finally after the  modifications has been done we rebuild the kernel using
make -j$(nproc)
sudo make modules_install
sudo make install
•	Reboot the kernel 
•	After rebooting the kernel we ssh into the inner VM using :
sudo qemu-system-x86_64 -enable-kvm -hda jammy-server-cloudimg-amd64.img -m 512 -nographic(or whichever image file we are using for inner VM )
•	We can open another terminal of the outer VM while the inner VM is running and use the command : 
                     dmesg | grep "KVM Exit"  
to check for exits and the counts.

Step  5:

•	Then we check the status using 
                                         git status
•	We can see that we have made changes to vmx.c
•	Next we can use git add . to add it to the repository
•	And after adding it we can use 
   git commit -m "Implemented KVM exit tracking functionality"
to commit it to the repository
•	Finally we push the code using git push origin master to add it to the repository and save it in there.

3.	Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there more exits performed during certain VM operations? Approximately how many exits does a full VM boot entail?

Ans. During the boot process of the inner VM running within the outer VM, I noticed that the frequency of exits does not increase at a steady rate. 
Instead, there are periods where the number of exits spikes, especially during certain operations that involve heavy interaction with the hypervisor. 
For instance, during hardware initialization and memory mapping, the inner VM generates a significant number of exits. This is because the VM frequently makes CPUID queries to check CPU features, writes to model-specific registers through MSR_WRITE, and handles address translations using EPT_VIOLATION. 
These stages are resource-intensive, which leads to an increase in the number of exits.

Once the inner VM has completed these initialization tasks and begins its normal operations, the frequency of exits starts to stabilize. 
Most of the exits occur during the earlier stages of the boot process, and the number of exits drops significantly as the VM continues running in its regular state.

On average, a full VM boot generates thousands of exits. Lightweight operating systems might generate around 5,000 to 10,000 exits, while more complex systems could see upwards of 50,000 exits. 
This demonstrates that the boot process is a particularly exit-heavy phase, primarily due to the frequent CPUID, MSR_WRITE, and EPT_VIOLATION exits that happen during hardware setup and memory management.


4.	Of the exit types, which are the most frequent? Least?
Ans. The most frequent exit types I observed are CPUID, MSR_WRITE, and EPT_VIOLATION. CPUID exits occur often as the inner VM queries the CPU features during its boot process. 
MSR_WRITE exits are common because the inner VM frequently writes to model-specific registers (MSRs) to configure its processor. EPT_VIOLATION exits also happen frequently as the inner VM handles memory mapping and translates virtual addresses to physical addresses.

On the other hand, the least frequent exit types are related to exceptional or error conditions, such as TRIPLE_FAULT or HLT.
 These types of exits are rare and typically happen when the inner VM encounters critical errors or halts. 
Since these conditions don’t occur frequently during normal operations or the boot process, these exit types are much less common compared to the others involved in hardware initialization and memory management.

![App Screenshot](https://github.com/PratikKM2002/linux/blob/master/screenshots/Screenshot%20(636).png)







