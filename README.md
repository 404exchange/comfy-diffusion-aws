# comfy-diffusion-aws
A fork from mikeage@ 's stable-diffusion-aws, with a goal of running comfyui with the same on-demand spot instance setup. 

For now, the focus will be on getting a functional automatic111 webui that can be called on demand, attach to permanent storage with the correct checkpoints and LORAs, and attach to an existing Elastic Network Adapter at eth0 to integrate into 404.exchange's existing services. 

I will also change the targeted instance type to g4dn.2xlarge and maximum bid price for spot instance to 40 cents. The g4dn.2xlarge instance recently averages about 31 cents an hour through spot instance pricing. 
