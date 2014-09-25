FIFO
====

A simple library multiplatform to handle named pipe, works like File.

Reader Example:
    pipe = Fifo.new('/path/to/file') #non-blocking
    # OR
    # pipe = Fifo.new('/path/to/file', :r, :wait) #blocking

    pipe.read(2)
    pipe.getc
    pipe.gets
    pipe.readline

Writer Example:
    pipe = Fifo.new('/path/to/file', :w, :nowait) #non-blocking
    # OR
    # pipe = Fifo.new('/path/to/file', :w, :wait)

    pipe.write "HI"
    pipe.print "X"
    pipe.puts "OH", "HAI"


============================

####Example:  Creating a Debug Interface to a ruby Application
* add a debug module
* insert debug Callback runDebug() into serial_api -  todo: create a Debug interface to inherit from
* main allocate debug module debugMgr
*   add the serial_api object to the debugMgr tag as 'serial'
* last, create debug script file for command line CLI to inject our debugs.
* 

```

##########################
require 'logger'
require 'fifo'



# DebugUtil Class
#  method: processFile
class DebugUtil
  
  
  # # # # # # # # # #
  # initialize() DebugUtil
  #
  def initialize(filename,logging)
    @debugfile = filename
    @logger = logging
    @logger.info("debug manager initialized..")
    @debugs = Hash.new()
  end

  #
  # addDebug(subsystem,object)
  #   - Add a subsystem to call when a Debug command is found
  #   - object is expected to implement its debugs runDebug(parms) parms[0]=cmd
  #
  def addDebug(subsystem,object)
    if subsystem.kind_of?(String)
      @debugs[subsystem] =  object 
      @debugs.each { |key,value| puts "#{key}, #{value}" } 
    else
      puts "addDebug(subsytem,object) subsystem not a String"
    end
    
  end
  
  def removeDebug()
    
  end
  
  # -open fifo for read  
  # -readline, split string cmd,parms
  # -find "subsystem" if exists call runDebug(args)
  def processFile()

    begin
      @debug_thread = Thread.new do
        #if fifo notExist then create pipe = Fifo.new('/tmp/ccmdebug')
        if $pipe.nil?
          $pipe = Fifo.new("/tmp/ccmdebug", :r, :wait) #blocking
        end
        #puts $pipe.methods
          
        loop do 
          msg= "Debug: debug.processFile()"+Time.now.strftime("%d/%m/%Y %H:%M:%S")
          @logger.info msg
          puts (msg)
          count = 0

          if !$pipe.nil?
            begin 
              line = $pipe.readline()
            rescue => e
              puts e
              line = nil
              $pipe.close
              $pipe = Fifo.new("/tmp/ccmdebug", :r, :wait) #blocking
            rescue SystemExit, Interrupt
              $pipe.close
              raise
            end
          end
          
          if (line != nil) 
            puts line
            args = line.strip.split(',', 2)
            if args.count > 0
      
              cmd = args.shift
              @debugs.each { |key, value|
                 puts "for each debug #{key},#{value} test #{cmd}.casecmp(#{key})"
                 if cmd.casecmp(key) == 0
                   puts " key found #{key}"
                   if value.respond_to?(:runDebug,include_all=true)
                     puts "runDebug found calling #{value}.runDebug(#{args.to_s}) " 
                     value.runDebug(args)
                     break
                   else
                     puts ":runDebug not found"
                   end
                 end
              }
            
            end # if count      
          end # !nil
        end #loop
      end # end thread.run

    rescue => e
      @logger.error "DebugAPI: debug processFile thread crashed. #{e}"
    end
      
  end # processFile


end # class
================================

# Application - demo
# declare debug Manager
$debugMgr = DebugUtil.new("debug.txt",logger)
$debugMgr.addDebug("serial", $serial_api)
$debugMgr.processFile()

# Serial_api
 #
  public
  def runDebug(args)
    @logger.debug("runDebug: #{args}, kindof Array=#{ (args.kind_of?(Array)) ? "true" : "false"}")
    if args.kind_of?(Array)
      if args.length == 1
        buffer = args.shift.split(',')
      end

      cmd = buffer.shift
      puts "runDebug() #{cmd}"
      
      if cmd.casecmp("reset") == 0 
        self.serialreset()
      elsif cmd.casecmp("loglevel") == 0
        self.set_log_level(buffer)
      elsif cmd.casecmp("send") == 0
        self.send_command(buffer.join('/')+"\n")
      elsif cmd.casecmp("newData") == 0
        newDataStructures()
      elsif cmd.casecmp("dump") == 0
        dumpDataStructures()
      end
    end
  end



=============================================

#Command line inject debug commands into application.

#!/usr/bin/env ruby

require 'fifo'


## Debug Interface
# fifo input to ccm_daemon.rb
#  currently only serial has a runDebug() method and commands 
# example: 
#      ./debug subsystem cmd [params]
#      ./debug serial reset
#      ./debug serial loglevel 0
#      ./debug serial send locktalk lock1 open ; sleep 3 ; ./debug serial send locktalk lock1 close
# 
#
# see:  cc_debug_util.rb

# 
# tail -f log/ccm_daemon.log to monitor output.

ARGV.each do|a|
  puts "Argument: #{a}"
end

pipe = Fifo.new("/tmp/ccmdebug", :w, :wait)
puts pipe

puts ARGV.class.name

pipe.puts ARGV.join(",")

```
