#VHDL Naming Rules



###Constants and Generic Parameters  
are full _UPPERCASE_, underscores allowed.  For expressing some 'physical' values, like  e.g. 'Hz', you can exceptionally use some lowercase  
	e.g. ```WIDTH_D , NUM_PIXELS , BASE_FREQUENCY_in_kHz, DELAY_TIME_in_clocks, CLOCK_PERIOD_in_ps```

###Pin names (aka Port names)  
are _MixedCase_, use _n to indicate active low in- or outputs, else no other underscores are allowed. They should as much as possible match the name in the schematic, if these are too cryptic talk to the schematic designer to change them.  
	e.g  ```DPixQ , WrFree , SlWr_n```

**Internal signal** names have  two formats:
   1. When a signal is used to connect a component to another component or to internal glue logic the signal name has the format: _lowercase_MixedCase_ where _lowercase_ is the name of the component generating the output signal, the underscore ('_') serves as concatenation and the _MixedCase_ is the name of the pin feeding that signal. The underscore is analogous to the decimal point operator as used in AHDL where the (automatic) signal is made up as follows: lowercase.MixedCase  
e.g.  ```synchronisebypass_Q , lcodecntr_Q```

   2. When a signal is effectively 'glue logic'  or a local variable it is written in full _lowercase_, with no underscores allowed. The plain _lowercase_ may complicate making good names a bit, but the idea is that you avoid _glue logic_ in the upper layer files altogether anyway and that you don't need many in the lower layer modules. If a local signal is driving a pin on a component the signal's name is a concatenation of the driven component's name and the driven component pin's name (without underscores).  
e.g.  ```lcodecntrcnten```

###Procedure and Function names  
are in full _lowercase_ also,  but with underscores allowed. This conflicts somewhat with the internal signal naming rule but the necessary parenthesis show the difference.

###Structured Element names  
start with a single lowercase letter indicating their 'type' 
   * 'r' : record
   * 'a' : array
   * 'e' : enum
   * 't' : subtypes like ```t_ipnumberval is natural range o to 255``` Use of subtypes is however discouraged, unless they are really meaningful and enhance the readability  
   
separated by an underscore from the actual name in full _lowercase_. If a Structured Element is made from other Structured Elements, the prefixes are stacked. Arrays have their dimension added dirrecty after the ```a``` e.g.:  
   * ```r_framedoutput```
   * ```r_ipnumber```
   * ```t_ipnumberval```
   * ```a4_t_ipnumberval```
   * ```a16_r_ipnumber```
   * ```a16_a4_t_ipnumberval```
  
The signal, port and variable names given to such Structured Elements follow the above rules. Functions to convert 'from and to' other types start with 'to_' followed by the general type of the destination, e.g.:  
   * ```to_std_logic_vector( r : r_framedoutput ) return std_logic_vector is```
   * ```to_record( v: std_logic_vector ) return r_framedoutput is ```
   * ```to_array( v: std_logic_vector ) return a_t_ipnumberval is ```

Indentation and proper placement of comments is also important

Warning: VHDL is case-insensitive! It is however recommended to write VHDL-reserved words (like library, use, port, begin ...) in lowercase, possibly with the first letter capitalized, but never in full uppercase. The syntax coloring in modern editors does a perfectly good job!


A short example (of a lowest level block):

```VHDL
-- "JPEG ADV212"
--
-- Taker / Provider style interface to the ADV212 JPEG in encoder mode, raw
-- pixel - jdata mode
-- VData[] , VFrm , VStrb and VRdy to the input side of ADV212
-- JData[] , Hold and Valid from the output side of the ADV212

-- 25-01-2008 jb
-- initial creation

library ieee;
use IEEE.Std_Logic_1164.all;
use IEEE.numeric_std.all;
use IEEE.Math_real.all ;

library lpm;
use lpm.lpm_components.all;

library altera_mf; 
use altera_mf.altera_mf_components.all; 

library cccs ;
use cccs.cccs.all ;

entity jpegadv212 is
	generic (
		WIDTH_D		: integer := 10 ;	-- can be 8, 10  or 12, i.e. as wide
										-- as delivered by sensor
		WIDTH_Q		: integer := 16 ;	-- can be 8 or 16
		WIDTH_V		: integer := 10 ;	-- can be 8, 10  or 12, i.e. As
										-- accepted by the ADV212, cannot be
										-- more than the input data
		WIDTH_J		: integer := 8 		-- the ADV212 always returns 8-bit
										-- data (at least in the mode we are
										-- using it)
		) ;

	port(
		Clk		: in	 std_logic ; 									-- same clock for in- and out-put
		
		D		: in	 std_logic_vector(WIDTH_D - 1 downto 0) ;
		DPixQ	: in	 std_logic_vector(2 downto 0) ;
		WrFree	: out std_logic ;
		WrEn	: in	 std_logic ;
		
		Q		: out std_logic_vector(WIDTH_D - 1 downto 0) ; 		
		QPixQ	: out std_logic_vector(2 downto 0) ;
		RdAvail	: out std_logic ;
		RdEn	: in	 std_logic ;
		
		ByPass	: in	 std_logic ;									-- default '0' is no-ByPass
				
		DJ		: out std_logic_vector(WIDTH_V - 1 downto 0) ; 	-- Vdata bus
		VFrm	: out std_logic ;
		VRdy	: in	 std_logic ;
		VStrb	: out std_logic ;
		
		SComm4	: in	 std_logic := '0' ; 							-- end of jpeg-tile, valid for 4 									 
																		-- last bytes coming from ADV212
		
		QJ		: in	 std_logic_vector(WIDTH_J - 1 downto 0) ;	-- Jdata bus
		Valid	: in	 std_logic ;
		Hold	: out std_logic 
		 		
		);
	end jpegadv212;




architecture a of jpegadv212 is

	component synchroniser
		generic (
			NUM_STAGES : integer := 2
			) ;
		port (
			Clk		: in  std_logic ;
			D		: in  std_logic ;
			Q		: out std_logic
			) ;
		end component ;



	signal synchronisebypass_Q	: std_logic ;
	signal lcodecntrcnten		: std_logic ;
	signal lcodecntr_Q			: std_logic_vector(1 downto 0) ;


begin

	assert WIDTH_J = 8 
	report "ADV212 only returns 8 bit data in this mode" 
	severity error ;

	assert WIDTH_D >= WIDTH_V
	report "Supplied input data must have width greater then or equal to width 			of data sent to ADV212" 
	severity error ;


	-- synchronize some inputs ...
	synchronisebypass : synchroniser
		port map(
			Clk		=> Clk ,
			D		=> ByPass ,
			Q		=> synchronisebypass_Q
			) ;


	lcodecntr : lpm_counter
		generic map (
			LPM_WIDTH => 2 ,    -- MUST be greater than 0
			LPM_TYPE  => "LPM_COUNTER" 
			)
		port map (
			Clock	=> Clk ,
			Cnt_En	=> lcodecntrcnten ,
			Q 		=> lcodecntr_Q
			) ;
		
		

	-- everything is combinatorial over here

	process ( synchronisebypass_Q , DPixQ , VRdy , WrEn , d , RdEn , Valid , 
				QJ , SComm4 , lcodecntr_Q )
		begin
			VFrm				<= '0' ;
			VStrb				<= '0' ;
			Hold				<= '0' ;
			DJ					<= CONV_STD_LOGIC_VECTOR(0 , WIDTH_V) ;
			QPixQ				<= CONV_STD_LOGIC_VECTOR(0 , 3) ;
			Q					<= CONV_STD_LOGIC_VECTOR(0 , WIDTH_D) ;
			lcodecntrcnten		<= '0' ;
		
			if (synchronisebypass_Q = '0' ) then
				-- use jpeg
				if (DPixQ = PIXEL_BOF) then
					VFrm <= '1' ;
				end if ;
				WrFree <= VRdy ;
				if (VRdy = '1') then
					VStrb <= WrEn ;
				end if ;
				DJ <= D(WIDTH_D - 1 downto WIDTH_D - WIDTH_V) ;
				Hold <= not RdEn ;
				RdAvail <= Valid ;
				if (RdEn = '1') and (Valid = '1') and (SComm4 = '1') then
					lcodecntrcnten <= '1' ;
					if (lcodecntr_Q = 3) then
						QPixQ <= CONV_STD_LOGIC_VECTOR(PIXEL_EOF , 3)  ;
					end if ;
				end if ;		
				Q(WIDTH_D-1 downto WIDTH_D - WIDTH_J) <= QJ ;	-- left align
														-- for later
			else
				-- ByPass
				WrFree <= RdEn ;
				RdAvail <= WrEn ;
				Q <= D ;
				QPixQ <= DPixQ ;
			end if ;
		end process ;
end a;
```

