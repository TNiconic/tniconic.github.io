---
layout: post
title: Utilizing Zeek Intel
date: 2022-06-23 12:00:00 -500
categories: [zeek,intelligence,automation,analytics]
tags: [zeek,analytics,python,securityonion,analytics]
---

# Intro to Zeek Intel

![Repo](/assets/zeekintel/zeek.png)

Hello! The main focus of this blog post is going to be about Zeek Intelligence. I'm writing this as it's something that I've been requested to setup and configure before and is one of the most utilized/helpful features. Today I'll be discussing the following:

1. What Zeek Intelligence is.
    
2. How it can benefit organizations.

3. How to utilize Zeek Intel inside SecurityOnion2

4. Practical example.

# What is Zeek Intel?
Zeek Intelligence is a framework created by Zeek that allows admins to configure alerts based off intelligence (indicators) against live network traffic. In short, you configure Zeek to look out for indicators while ingesting network traffic that will generate alerts that analysts can query. For further reading you can read their very helpful docs here:

 https://docs.zeek.org/en/master/frameworks/intel.html

In laymans terms, this capabilty allows you to pattern match known IOC's and give you visual alerts that a match has been detected. This comes at the cost of some performance degredation but I've found is well worth the tradeoff.

![Repo](/assets/zeekintel/diagram.png)

# Why is this beneficial?
I've been in many SOC environments and I've come to find that the most valuable resource is time. With that in mind, Zeek Intel allows us to not have to spend cycles looking for known IOC's or private intelligence. Instead, we can input those static analytics into our pre-existing framework and have it do the work for us; pretty neat! In practice this looks like the following. Your organization either has closed-source intelligence or you import open source knoweldge. Then you configure your SIEM that's already utilizing Zeek to be matching incoming traffic with said intelligence. This then gives your analysts a platform to quickly and effectively monitor those alerts to see if there has been a hit. Below is an example dashboard of a scenario where intelligence was used to indicate a malicious IP address. Once this address appeared in the network traffic we were able to see it instantly and were able to dive deeper and eventually take action.

![Repo](/assets/zeekintel/example3.png)

# Prerequisites
The real only true prerequisite required is to have Zeek ingesting traffic for you. However one of the big downsides with Zeek Intel is the formatting required to get it working. Normally this isn't a big issue, but when you have potentially tens of thousands of intelligence indicators you want referenced it becomes overwhelming. That's why I tweaked a foratting script I found to make it compatible with SecurityOnion2. This new script will take CSV files and format them accordingly to match with Zeek Intel. The script will be provided below along with a copy of a slide-deck presentation I've given to cyber professionals to implement this solution in their deployments. The last thing to be congnisant of is the formatting of your intelligence. In order for Zeek to be able to properly and efficiently check your network traffic against your intelligence you will have to properly define your various indicators. Listed below is a link to the various indicator types that Zeek supports as well as a practical example I created showing various csv files I later imported.

Supported Zeek Intel Indicators: https://docs.zeek.org/en/master/scripts/base/frameworks/intel/main.zeek.html

List of formatted CSVs: 

![Repo](/assets/zeekintel/example4.png)

# Utilizing Zeek Intel with SO2
Once you're ready to implement this into your environment, first you need to push your list of intelligence in csv format and the provided script below into your SIEM. The following instructions were done using SecurityOnion2. For examples of intelligence you can use to follow along you can use the link listed below which is a repository containing csv files with various IOC types.

https://osint.digitalside.it/Threat-Intel/csv/
 
 Again, the following instructions are done using the provided script and closed source information imported into a SecurityOnion2 SIEM. Once you have them imported you will need to rename the script to ensure it ends with .py and give if full permissions.

 ```bash
sudo mv zeek_importer.txt zeek_importer.py
```
```bash
sudo chmod 777 zeek_importer.py
 ```

Then you will want to create a temporary intel.dat file to later append your indicators to.

```bash
sudo cp /opt/so/saltstack/default/salt/zeek/policy/intel/* /opt/so/saltstack/local/salt/zeek/policy/intel/
```
You will now have an intel.dat file located in the above directory. This is imporant as we essentially copied over files from a reference directory into our SecurityOnion2 production directory. Now you're ready to start using the script! Zeek intel uses different   

# Practical Example
One of the most useful intel types to implement is md5 checking. If configured, Zeek will automatically detect files and hash them. You can then use Zeek Intelligence to cross-reference any incoming files against malicous md5 hashes. Let's get started!

First: You will want to run the script.
```bash
sudo python3 zeek_importer.py
```

Next, you will need to answer the following questions. I will list out each question and give context to what you should put.

**Indicator type:**
Here you will want to put the indicator type as listed by Zeek in the link provided above.

**Source:**
If pulling this from the internet or internal, you may want to reference that here.

**Description:**
Essentialy a summary of what this alert is trying to capture.

**Absolute Path of the input file:**
You will want to point this to the csv of the appropriate type you uploaded earlier.

**Absolute Path of the Output file:**
You will want to put the file into the /opt/so/saltstack/local/salt/zeek/policy/intel/ folder as a .dat file that you will later append to the intel.dat file in the same directory.

![Repo](/assets/zeekintel/example1.png)

Next change into the aformentioned direcotry and you will append that file.

```bash
cd /opt/so/saltstack/local/salt/zeek/policy/intel/
```

```bash
sudo cat md5.dat >> intel.dat
```

If you now cat the contents of your intel.dat file it should look similar to below.

![Repo](/assets/zeekintel/example2.png)

The last steps in order to get it working are to restart some services. 

```bash
sudo so-zeek-restart
```

```bash
sudo so-kibana-restart
```

If doing this on a distributed SO2 deployment, you will want to also push this update to your forward nodes.

```bash
sudo salt "*" state.highstate
```
And ther you go! You should now have a fully functioning Zeek Intel deployment that captures all the indicators listed in your md5.dat created file. For other indicators you would just repeat these steps for each indicator type you want to import.

# Resources

[Professional Slide Deck](/assets/zeekintel/Blog_Zeek_Intel_2.pptx)

**Python Code for Formatting Zeek Intelligence:**

```Python3
#!/usr/local/bin/python3

import sys, os
from os import path

# prompts are suppressed.
def main(args, use_input = True):

    # Strip the filename from the args
    args.pop(0)

    # Check to see if arguments are being passed
    if len(args) > 0:
        use_input = False

        # Check to make sure all arguments are present
        if len(args) < 5:
            print('')
            print('Only '+str(len(args))+' arguments were present but 5 are required.')
            print('Please check the README for complete usage instructions.', end='\n\n')
            return

    # If input is True, prompt the user for the required information
    if not use_input:
        indicator_type = args[0]
        source = args[1]
        description = args[2]
        input_file = args[3]
        output_file = args[4]

    # Else, set the required data based on the passed arguments
    else:
        print('')
        indicator_type = input('Enter the indicator type : ')
        source = input('Enter the source : ')
        description = input('Enter the description : ')
        input_file = input('Enter the absolute path for the input file : ')
        output_file = input('Enter the absolute path for the output file : ')

    # Check to see if the input file exists
    if not path.exists(input_file):
        print('')
        print('Can\'t read from the input file.')
        print('Does it exist? Do you have read permission?', end='\n\n')
        return

    # Try to open the input and output files
    try:
        with open(input_file, 'r') as infile, open(output_file, 'w') as outfile:

            # Write the heading line for the intel feed file
            outfile.write('#fields\tindicator\tindicator_type\tmeta.source\tmeta.do_notice\tmeta.desc\n')

            # For each line in the input file, write a tab-delimited line to the
            # output file formatted
            for line in infile:
                outfile.write(line.strip()+'\tIntel::'+str(indicator_type)+'\t'+str(source)+'\tT\t'+str(description)+'\n')

        infile.close()
        outfile.close()

    except IOError:
        print('')
        print('Can\'t write to the output file.')
        print('Do you have write permission for the path specified?', end='\n\n')
        return

    finally:
        print('')
        print('Finished!')
        print('You can find your output file here:')
        print(output_file, end='\n\n')
        return

# Execute the main function
if __name__ == '__main__' :
    main(sys.argv)
```