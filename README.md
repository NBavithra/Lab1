LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
USE ieee.numeric_std.ALL;

--
-- 7-segment display driver. It displays a 4-bit number on 7-segments 
-- This is created as an entity so that it can be reused many times easily
--

ENTITY SevenSegment IS PORT (
   
   dataIn      :  IN  std_logic_vector(3 DOWNTO 0);   -- The 4 bit data to be displayed
   blanking    :  IN  std_logic;                      -- This bit turns off all segments
   
   segmentsOut :  OUT std_logic_vector(6 DOWNTO 0)    -- 7-bit outputs to a 7-segment
); 
END SevenSegment;

ARCHITECTURE Behavioral OF SevenSegment IS

-- 
-- The following statements convert a 4-bit input, called dataIn to a pattern of 7 bits
-- The segment turns on when it is '0' otherwise '1'
-- The blanking input is added to turns off the all segments
--

BEGIN

   with blanking & dataIn SELECT --  gfedcba        b3210      -- D7S
      segmentsOut(6 DOWNTO 0) <=    "1000000" WHEN "00000",    -- [0]
                                    "1111001" WHEN "00001",    -- [1]
                                    "0100100" WHEN "00010",    -- [2]      +---- a ----+
                                    "0110000" WHEN "00011",    -- [3]      |           |
                                    "0011001" WHEN "00100",    -- [4]      |           |
                                    "0010010" WHEN "00101",    -- [5]      f           b
                                    "0000010" WHEN "00110",    -- [6]      |           |
                                    "1111000" WHEN "00111",    -- [7]      |           |
                                    "0000000" WHEN "01000",    -- [8]      +---- g ----+
                                    "0010000" WHEN "01001",    -- [9]      |           |
                                    "0001000" WHEN "01010",    -- [A]      |           |
                                    "0000011" WHEN "01011",    -- [b]      e           c
                                    "0100111" WHEN "01100",    -- [c]      |           |
                                    "0100001" WHEN "01101",    -- [d]      |           |
                                    "0000110" WHEN "01110",    -- [E]      +---- d ----+
                                    "0001110" WHEN "01111",    -- [F]
                                    "1111111" WHEN OTHERS;     -- [ ]

END Behavioral;

--------------------------------------------------------------------------------
-- Main entity
--------------------------------------------------------------------------------
--DECLARING THE VARIABLES ALREADY FOUND WITHIN THE LIBRARY (DECLARING WE WANT TO USE THEM)
LIBRARY ieee;
USE ieee.std_logic_1164.ALL;
USE ieee.numeric_std.ALL;

ENTITY Lab4 IS
   PORT(
      
      clock_50   : IN  STD_LOGIC;
 
      ledr       : OUT STD_LOGIC_VECTOR(11 DOWNTO 0); -- LEDs, many Red ones are available
      ledg       : OUT STD_LOGIC_VECTOR( 8 DOWNTO 0); -- LEDs, many Green ones are available
      hex0, hex2 : OUT STD_LOGIC_VECTOR( 6 DOWNTO 0)  -- seven segments to display numbers
);
END Lab4;

ARCHITECTURE SimpleCircuit OF Lab4 IS

--
-- In order to use the "SevenSegment" entity, we should declare it with first
-- 

--SEVEN SEGMENT DISPLAY
   COMPONENT SevenSegment PORT(        -- Declare the 7 segment component to be used
      dataIn      : IN  STD_LOGIC_VECTOR(3 DOWNTO 0);
      blanking    : IN  STD_LOGIC;
      segmentsOut : OUT STD_LOGIC_VECTOR(6 DOWNTO 0)
   );
   END COMPONENT;
----------------------------------------------------------------------------------------------------
   --DECLARING VARIABLES AND THEIR FUNCTION, WHETHER THEY CAN BE ADDED OR JUST USED AS AN AND/OR/XOR
   CONSTANT CLK_DIV_SIZE: INTEGER := 25;     -- size of vectors for the counters

   --SIGNAL Main1HzCLK:   STD_LOGIC; -- main 1Hz clock to drive FSM
   SIGNAL OneHzBinCLK:  STD_LOGIC; -- binary 1 Hz clock
   SIGNAL OneHzModCLK:  STD_LOGIC; -- modulus1 Hz clock
   SIGNAL TenHzModCLK:     STD_LOGIC; -- 10 Hz clock

   SIGNAL bin_counter:  UNSIGNED(CLK_DIV_SIZE-1 DOWNTO 0) := to_unsigned(0,CLK_DIV_SIZE); -- reset binary counter to zero
   SIGNAL mod_counter_1:  UNSIGNED(CLK_DIV_SIZE-1 DOWNTO 0) := to_unsigned(0,CLK_DIV_SIZE); -- modulus counter for ten hertz
   SIGNAL mod_counter_10: UNSIGNED(CLK_DIV_SIZE-1 DOWNTO 0) := to_unsigned(0,CLK_DIV_SIZE); --reset modulus counter to zero
   SIGNAL mod_terminal_1:  UNSIGNED(CLK_DIV_SIZE-1 DOWNTO 0) := to_unsigned(0,CLK_DIV_SIZE);
   SIGNAL mod_terminal_10:  UNSIGNED(CLK_DIV_SIZE-1 DOWNTO 0) := to_unsigned(0,CLK_DIV_SIZE);
   
   TYPE STATES IS (GLEDNS_FLASH,GLEDNS_SOLID, AMBERNS, GLEDEW_FLASH, GLEDEW_SOLID, AMBEREW);   -- list all the STATES
   SIGNAL state, next_state:  STATES;                 -- current and next state signals od type STATES
   SIGNAL state_number: STD_LOGIC_VECTOR(3 DOWNTO 0); -- binary state number to display on seven-segment
   SIGNAL state_counter: UNSIGNED(3 DOWNTO 0);        -- binary state counter to display on seven-segment
----------------------------------------------------------------------------------------------------
--ASSIGNING VARIABLES TO AN ITEM ON FPGA BOARD IN THE CASE LEDG(2) IS KNOWN TO BE ONE HZ BINARY COUNTER 
BEGIN

  -- BinCLK: PROCESS(clock_50)
   --BEGIN
    ---  IF (rising_edge(clock_50)) THEN -- binary counter increments on rising clock edge
       --  bin_counter <= bin_counter + 1;
    -- END IF;
  -- END PROCESS;
  -- OneHzBinCLK <= std_logic(bin_counter(CLK_DIV_SIZE-1)); -- binary counter MSB
  -- LEDG(2) <= OneHzBinCLK;
----------------------------------------------------------------------------------------------------
--THE SWITCHES AND HOW THEY INCREASE/DECREASE FREQUENCY
--ASSIGNING ONE HZ MOD CLOCK TO LEDG(1)

      mod_terminal_1 <= "1011111010111100000111111" ;  -- F =   1 Hz, T/2 = 25000000 * T0             
      mod_terminal_10 <= "0001001100010010110011111";  -- F =  10 Hz, T/2 =  2500000 * T0
                 
--THE CLOCK ENTERS THE IF STATEMENT THEN GETS INCREMENTED UNTIL IT REACHES THE SAME FREQUENCY AS WHICH IS SWITCHES ARE PLACED (MOD TERMINAL)
--THEN ONCE ITS THE SAME, THE VALUE FOR THE ONE HZ MOD CLK WILL BE FLIPPED AND THE COUNTER IS RESET AGAIN
--THE FLIPPED ONE IS THE ONE THAT GETS OUTPUTTED LEDG(1)

   ModCLK1: PROCESS(TenHzModCLK) 
   BEGIN
      IF (rising_edge(TenHzModCLK)) THEN -- modulus counter increments on rising clock edge
         IF (mod_counter_1 = mod_terminal_1) THEN       -- half period
            OneHzModCLK <= NOT OneHzModCLK;                 -- toggle
            mod_counter_1 <= to_unsigned(0,CLK_DIV_SIZE); -- reset counter
         ELSE
            mod_counter_1 <= mod_counter_1 + 1;
         END IF;
      END IF;
      END PROCESS; 
     
   ModCLK10: Process(clock_50)
   BEGIN
      IF (rising_edge(clock_50)) THEN -- modulus counter increments on rising clock edge
         IF (mod_counter_10 = mod_terminal_10) THEN       -- half period
            TenHzModCLK <= NOT TenHzModCLK;                 -- toggle
            mod_counter_10 <= to_unsigned(0,CLK_DIV_SIZE); -- reset counter
         ELSE
            mod_counter_10 <= mod_counter_10 + 1;
         END IF;
      END IF;
   END PROCESS;
   
   LEDG(0) <= TenHzModCLK;
   LEDG(1) <= OneHzModCLK;
----------------------------------------------------------------------------------------------------

  
  
--SWITCH CONTROLS THE STATES, AND DEPENDING ON THE SWITCH, THE NUMBER WILL BE SHOW ON ONE OF THE HEX
   FSM: PROCESS(state) -- main FSM
   BEGIN
--      next_state <= state; 	-- The only purpose of this line is to give initial value to the signal 'next_state' in order to avoid latch creation. 
      CASE state IS
         WHEN GLEDNS_FLASH =>
            state_number <= "0000";
            ledr(0) <= '1';
            ledg(7) <= '0';
            ledg(8) <= TenHzModCLK;
            ledr(11) <= '0';
            IF (state_counter = "0010") THEN 
               next_state <= GLEDNS_SOLID;
            ELSE
               next_state <= GLEDNS_FLASH;
            END IF;
         WHEN GLEDNS_SOLID =>
            state_number <= "0001";
            ledr(0) <= '1';
            ledg(7) <= '0';
            ledg(8) <= '1';
            ledr(11) <= '0';
            IF (state_counter = "0110") THEN 
               next_state <= AMBERNS;
            ELSE
               next_state <= GLEDNS_SOLID;
            END IF;
         WHEN AMBERNS =>
            state_number <= "0010";
            ledr(0) <= '1';
            ledg(7) <= '0';
            ledg(8) <= '1';
            ledr(11) <= TenHzModCLK;
            IF (state_counter = "1000") THEN 
               next_state <= AMBERNS;
            ELSE
               next_state <= GLEDNS_SOLID;
            END IF;
         WHEN GLEDEW_FLASH =>
            state_number <= "0011";
            ledr(0) <= '0';
            ledg(7) <= TenHzModCLK;
            ledg(8) <= '0';
            ledr(11) <= '1';
            IF (state_counter = "1010") THEN 
               next_state <= GLEDEW_SOLID;
            ELSE
               next_state <= GLEDEW_FLASH;
            END IF;
         WHEN GLEDEW_SOLID =>
            state_number <= "0100";
            ledr(0) <= '0';
            ledg(7) <= '1';
            ledg(8) <= '0';
            ledr(11) <= '1';
            IF (state_counter = "1110") THEN 
               next_state <= AMBEREW;
            ELSE
               next_state <= GLEDNS_SOLID;
            END IF;      
         WHEN AMBEREW => -- STATE3
            state_number <= "0101";
            ledr(0) <= TenHzModCLK;
            ledg(7) <= '0';
            ledg(8) <= '0';
            ledr(11) <= '1';
            IF (state_counter = "0000") THEN 
               next_state <= GLEDNS_FLASH;
            ELSE
               next_state <= AMBEREW;
            END IF;      
      END CASE;
   END PROCESS;
----------------------------------------------------------------------------------------------------

   SeqLogic: PROCESS(TenHzModCLK,OneHzModCLK) -- creats sequential logic to latch the state
   BEGIN
      IF (rising_edge(OneHzModCLK)) THEN
         
         state_counter <= state_counter + 1;                      -- on the rising edge of clock the current state is updated with next state
         
         IF (state /= next_state) THEN
            state_counter <= "0000";
                -- on the rising edge of clock the current counter is incremented if state is STATE1
         END IF;
         state <= next_state;
      END IF;
   END PROCESS;
----------------------------------------------------------------------------------------------------
   D7S0: SevenSegment PORT MAP( state_number, '0', hex0 );
   D7S4: SevenSegment PORT MAP( std_logic_vector(state_counter), '0', hex2 );

END SimpleCircuit;
