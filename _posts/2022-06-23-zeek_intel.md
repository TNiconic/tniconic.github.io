---
layout: post
title: Utilizing Zeek Intel
date: 2022-06-23 12:00:00 -500
categories: [zeek,intelligence,automation,analytics]
tags: [zeek,analytics,python,securityonion,analytics]
---

# Intro to Zeek Intel
![Repo](/assets/zeekintel/zeek.png)
Hello! The main focus of this blog post is going to be about Zeek Intelligence. I wanted to write this blog post as it's something that I've been requested to setup and configure for organizations before. For the sake of transparency, I wanted to write this blog post to help those out that aren't in my local area! Today I'll be discussing the following:
   1. What Zeek Intelligence is.
    
2. How it can benefit organizations.

3. How to utilize Zeek Intel inside SecurityOnion2

4. Practical example.

# What is Zeek Intel?
Zeek Intelligence is a framework created by Zeek that allows admins to configure alerts based off intelligence (indicators) against live network traffic. In short, you configure Zeek to look out for indicators while ingesting network traffic that will generate alerts that analysts can query. For further reading you can read their very helpful docs here: https://docs.zeek.org/en/master/frameworks/intel.html

![Repo](/assets/zeekintel/diagram.png)
# Why is this beneficial?
I've been in many SOC environments and I've come to find that the most valuable resource is time. With that in mind, Zeek Intel allows us to not have to spend cycles looking for known IOC's or private intelligence. Instead, we can input those static analytics into our pre-existing framework and have it do the work for us; pretty neat!

# Prerequisites
The real only true prerequisite required is to have Zeek ingesting traffic for you. However one of the big downsides with Zeek Intel is the formatting required to get it working. Normally this isn't a big issue, but when you have potentially tens of thousands of intelligence you want referenced it can become a hassel. That's why I tweaked a foratting script I found online to make it compatible with SecurityOnion2 that will take CSV files and format them accordingly. The script will be provided below along with a copy of a slide-deck presentation I've given to Cyber Professionals to implement this solution in their deployments.

# Utilizing Zeek Intel with SO2

# Resources

[Professional Slide Deck](/assets/zeekintel/Blog_Zeek_Intel_2.pptx)

**Python Code for Formatting:**

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