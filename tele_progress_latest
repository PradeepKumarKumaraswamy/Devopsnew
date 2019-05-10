puts "Subscription proc usage syntax: telemetry_pull  \"Protocol_list\" \"Dut_list\""
 
proc telemetry_pull { {proto all} {duts all} args} {
   global tele timefirst dut_list env gmi_global 
   stopwatch_init w1
   
   if { $proto == "all" } {
      set proto [list 'bgp','ospf','ldp']
   }
   
   if {$duts == "all"} {
      set duts $dut_list
   }

   set i 1
   
  
      foreach item $proto {   
         if { $item == "bgp" } {
            set path($i) "state router <Base> bgp neighbor <*> statistics last-established-time"
         } elseif { $item == "ospf" } {
            set path($i) "state router <Base> ospf <*> area <*> interface <*> neighbor-state <*> last-event-time"
         } elseif { $item == "isis" } {
            set path($i) "state router <Base> isis <*> interface <*> up-time"
         } elseif { $item == "ldp" } {
            set path($i) "state router <Base> ldp session <*> up-time"
         } else {
            puts "Invalid Protocol"
            return
         }
	     incr i
      }
	 
   set formatStr {%50s%50s}	
   foreach dut $duts {
      puts [format $formatStr "\n--------------***----------------" "-------------------***-----------------" ]
      puts "\n Asynchronous Subscription in Dut $dut"
	  if {[telemetry_subscribe path $dut stream$dut] == "FAILED"} {
	     set flag($dut) 1
		 regsub $dut $duts "" duts
		 if {[llength $duts] == 0 } {
		    return
	     }
	     continue
	  }
	  array set asynchronous_reply.$dut [telemetry_subscribe path $dut stream$dut]
	  set flag($dut) 0
   }
   
   puts "Recursive Polling"
   set i 1
   while {$i < 15} {
   
     foreach dut $duts {
        if {$flag($dut) == 0} {
           if {[telemetry_check $dut] == 1} {
		      set flag($dut) 1
	          regsub $dut $duts "" duts
	       } else {
		      continue
		   }
	    }	
     }
	  
	if {[llength $duts] == 0 } {
	   puts "Cleanup call executed for all streams"
	   telemetry_cleanup
	   return
	}
	
	sleep 5
    incr i	 		
   }
}


proc telemetry_subscribe { {path} {dut} {strName "stream0"} args} {
   upvar 1 $path pathnew
   
   catch {$dut getMgmtCommand "" -array pathnew -mode gmi -request stream -streamName $strName -process nonBlocking} err
   
   if {$err == "OK"} {
      $dut getMgmtCommand "" -array pathnew -mode gmi -request stream -streamName $strName -process nonBlocking
   } else {
      puts "NO SUBSCRIPTION DATA"
	  puts "$err"
	  return $err
   }
  
   unset path
   if { [info exist reply(count)]} {unset reply(count)}
   if { [info exist reply(state)]} {unset reply(state)}
   if { [info exist reply(ok)]} {unset reply(ok)}
   if { [info exist reply(state,detail)]} {unset reply(state,detail)}

   return [array get reply]
}










proc parse {reply} {
   upvar 1 $reply replytmp
   
   foreach n [array names replytmp] {
      if {[regexp {state router <Base> bgp neighbor <(\d+\.\d+\.\d+\.\d+|[\w:]+:+[\w:]+)> statistics (last-established-time)} $replytmp($n) match nbrip attribute]} {
		   set value [lindex $replytmp($n) end]
		   if {[string equal $attribute "last-established-time"]} {
		       regexp {\d+-\d+-\d+T\d+\:\d+} $value tim
               dict set tmp bgp $nbrip $attribute $tim
			} else {
		       dict set tmp bgp $nbrip $attribute $value
			}
	  } elseif {[regexp {state router <Base> ospf <(\d+)> area <(\d+\.\d+\.\d+\.\d+|[\w:]+:+[\w:]+)> interface <([a-zA-Z0-9 -/:_]*)> neighbor-state <(\d+\.\d+\.\d+\.\d+)> (last-event-time)} $replytmp($n) match id areaid intr nbrip attribute]} {
	       set value [lindex $replytmp($n) end]
		    if {[string equal $attribute "last-event-time"]} {
		       regexp {\d+-\d+-\d+T\d+\:\d+} $value tim
               dict set tmp ospf $id $areaid $intr $nbrip $attribute $tim
			} else {
		       dict set tmp ospf $id $areaid $intr $nbrip $attribute $value
			}
      } elseif {[regexp {state router <Base> isis <(\d+)> interface <([a-zA-Z0-9 -/:_]*)> (up-time)} $replytmp($n) match id intr attribute]} {
	       set value [lindex $replytmp($n) end]
		   dict set tmp isis $id $intr $attribute $value
	  } elseif {[regexp {state router <Base> ldp session <(\d+\.\d+\.\d+\.\d+\:\d+|[\w:]+:+[\w:\[\]]+)> (up-time)} $replytmp($n) match ip attribute]} {
	       set value [lindex $replytmp($n) end]
		   dict set tmp ldp $ip $attribute $value
	  } else {
	     continue
      }
   }
   if {[info exist tmp]} {
      return $tmp 
   } else { 
	  return 
   }
  
}

proc telemetry_cleanup {{duration long} {duts all} {verbose noisy} args} {
   global stb_globals dut_list env gmi_global
   
   if {$duts == "all"} {
      set duts $dut_list
   }
   
   foreach dut $duts {
      $dut streamHandler "" -streamName stream$dut -replyValues reply -request cleanup
   } 
}






proc printdata {{data} {protocol} args} {
   set formatStr {%40s%15s}
   puts [format $formatStr "Key" "Value"]
   puts $protocol
   set datanew [dict get $data $protocol]
   
   dict for {key value} $datanew {
     dict for {k2 v2} $value {
	   puts [format $formatStr $key:$k2 \t$v2]
	 }
   }
   
}




 

proc telemetry_write { {dut} {strName} args} {
   global dut_list env gmi_global
   
   $dut streamHandler "" -streamName $strName -replyValues reply -request get-pairs
   
   if {$reply(state) == "finished" } {
      puts "Subscription for $dut is complete"
      return [array get reply]
   } elseif { $reply(state) == "processing" } {
      puts "Subscription in progress"
      return [array get reply]
   } else {
      puts "Subscription Error "
	  return [array get reply]  
   }
}
  
 


 
proc telemetry_check { {dut} args} {
   global tele timefirst dut_list env gmi_global
   
      set formatStr {%50s%50s}
      puts [format $formatStr "\n--------------***----------------" "-------------------***-----------------" ]
      puts "\nDut in execution is $dut"
      if {[info exists gmi_global($dut.tele)]} {
	     array set reply [telemetry_write $dut stream$dut]
	     if {$reply(state) == "processing"} {
		    parray reply
		    return 0
	     } elseif {$reply(state) == "finished" } {		 
	        set gmi_global($dut.teletmp) [parse reply] 
	        #puts "Temporary subscription data"
	        #puts "[dict get $gmi_global($dut.teletmp)]\n"
	        set timecurrent [clock milliseconds]
			
	           dict for {protocol values} $gmi_global($dut.teletmp) { 
			
	              if {$protocol == "ospf"} {
			   
						if {[dict exists $gmi_global($dut.tele) $protocol]} {
	                    dict for {ospfid ospfvalue} $values {
                         dict for {areaid areavalue} $ospfvalue {
                          dict for {interface interface_value} $areavalue {
                           dict for {nbrip nbripvalue} $interface_value {
                            dict for {lastEstTime lastEstTime_value} $nbripvalue {
                             if {[dict get $gmi_global($dut.tele) ospf $ospfid $areaid $interface $nbrip $lastEstTime] != [dict get $gmi_global($dut.teletmp) ospf $ospfid $areaid $interface $nbrip $lastEstTime]} {
			                    puts "ospf '$ospfid $areaid $interface $nbrip $lastEstTime' Flapped from [dict get $gmi_global($dut.tele) ospf $ospfid $areaid $interface $nbrip $lastEstTime] to [dict get $gmi_global($dut.teletmp) ospf $ospfid $areaid $interface $nbrip $lastEstTime]"
				                dict set gmi_global($dut.tele) ospf $ospfid $areaid $interface $nbrip $lastEstTime [dict get $gmi_global($dut.teletmp) ospf $ospfid $areaid $interface $nbrip $lastEstTime]
                             } else {
						        dict create gmi_global($dut.tele) ospf $ospfid $areaid $interface $nbrip $lastEstTime [dict get $gmi_global($dut.teletmp) ospf $ospfid $areaid $interface $nbrip $lastEstTime] 
						     }
                        }}}}}
	   
	                } else  {
				        puts "New $protocol data appended"
				        dict append gmi_global($dut.tele) $protocol $values
                    }
					printdata $gmi_global($dut.tele) $protocol
			      } 
				  
			     if {$protocol == "ldp"} {
				 
			   
				    if {[dict exists $gmi_global($dut.tele) $protocol]} {
	                   dict for {ldpip ldpvalue} $values {
                        dict for {uptime uptimevalue} $ldpvalue {
				         set elapsed_time [expr {$timecurrent - $timefirst($dut.$protocol)}]
				         set totaltimeldp [expr {$elapsed_time + [dict get $gmi_global($dut.tele) ldp $ldpip $uptime]}]
                            if {[dict get $gmi_global($dut.tele) ldp $ldpip $uptime] > [dict get $gmi_global($dut.teletmp) ldp $ldpip $uptime] || $totaltimeldp > [dict get $gmi_global($dut.teletmp) ldp $ldpip $uptime]} {
			                   puts "ldp '$ldpip $uptime' Flapped from [dict get $gmi_global($dut.tele) ldp $ldpip $uptime] to [dict get $gmi_global($dut.teletmp) ldp $ldpip $uptime]"
				               dict set gmi_global($dut.tele) ldp $ldpip $uptime  [dict get $gmi_global($dut.teletmp) ldp $ldpip $uptime ] 
			                } else {
						       dict set gmi_global($dut.tele) ldp $ldpip $uptime  [dict get $gmi_global($dut.teletmp) ldp $ldpip $uptime ] 
						    }
 		               }}
	               } else  {
					  puts "New $protocol data appended"
				      dict append gmi_global($dut.tele) $protocol $values
                   }
				   printdata $gmi_global($dut.tele) $protocol
				   set timefirst($dut.$protocol) [clock milliseconds] 
	            }
				
				if {$protocol == "bgp"} {
				
				   if {[dict exists $gmi_global($dut.tele) $protocol]} {
	                  dict for {nbrip nbripvalue} $values {
                       dict for {lastEstTime lastEstTime_timevalue} $nbripvalue {
	                    if {[dict get $gmi_global($dut.tele) bgp $nbrip $lastEstTime] != [dict get $gmi_global($dut.teletmp) bgp $nbrip $lastEstTime]} {
			              puts "bgp '$nbrip $lastEstTime' Flapped from [dict get $gmi_global($dut.tele) bgp $nbrip $lastEstTime] to [dict get $gmi_global($dut.teletmp) bgp $nbrip $lastEstTime]"
				          dict set gmi_global($dut.tele) bgp $nbrip $lastEstTime [dict get $gmi_global($dut.teletmp) bgp $nbrip $lastEstTime] 
			            } else {
						   dict set gmi_global($dut.tele) bgp $nbrip $lastEstTime [dict get $gmi_global($dut.teletmp) bgp $nbrip $lastEstTime] 
						}
		              }}
	               } else  {
				      puts "New $protocol data appended"
				      dict append gmi_global($dut.tele) $protocol $values
                   }
				printdata $gmi_global($dut.tele) $protocol
				}
			}
			
		    return 1
			
		}	
	  		
     } else {
		puts "Initial poll"
        array set reply [telemetry_write $dut stream$dut]
	    
	    if {$reply(state) == "processing"} {
		   parray reply
		   return 0
	    } elseif {$reply(state) == "finished"} {
	       set gmi_global($dut.tele) [parse reply] 
	       dict for {protocol value} $gmi_global($dut.tele) {
	         set timefirst($dut.$protocol) [clock milliseconds]
		     printdata $gmi_global($dut.tele) $protocol
	       }
		   return 1
		} else {
     		puts "Failed polling" 
			return error
        }
     }
}

