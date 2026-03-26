library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;

entity traffic_fsm is
    generic (
        TICK_HZ     : integer := 10;
        MAIN_IS_NS  : integer := 1
    );
    port (
        clk, rst_n      : in  std_logic;
        tick            : in  std_logic;
        veh_ns, veh_ew  : in  std_logic;
        ped_pulse       : in  std_logic;
        emergency       : in  std_logic;
        night_mode      : in  std_logic;

        ns_g, ns_y, ns_r : out std_logic;
        ew_g, ew_y, ew_r : out std_logic;
        ped_walk, ped_dontwalk : out std_logic
    );
end entity;

architecture rtl of traffic_fsm is

    type state_t is (
        S_NS_G, S_NS_Y, S_ALL_RED1,
        S_EW_G, S_EW_Y, S_ALL_RED2,
        S_PED_WALK, S_PED_CLEAR,
        S_EMERG_ALL_RED, S_NIGHT_FLASH
    );

    signal state, next_state : state_t;

    signal ped_req_latched : std_logic := '0';

    -- Timer signals
    signal timer_count  : integer := 0;
    signal timer_limit  : integer := 0;
    signal timer_start  : std_logic := '0';
    signal timer_done   : std_logic := '0';
    signal timer_active : std_logic := '0';

    -- Time constants (seconds)
    constant T_GREEN_MIN : integer := 5 * TICK_HZ;
    constant T_YELLOW    : integer := 2 * TICK_HZ;
    constant T_ALLRED    : integer := 1 * TICK_HZ;
    constant T_WALK      : integer := 5 * TICK_HZ;

begin

--------------------------------------------------
-- Pedestrian latch
--------------------------------------------------
process(clk, rst_n)
begin
    if rst_n = '0' then
        ped_req_latched <= '0';
    elsif rising_edge(clk) then
        if ped_pulse = '1' then
            ped_req_latched <= '1';
        elsif state = S_PED_CLEAR then
            ped_req_latched <= '0';
        end if;
    end if;
end process;

--------------------------------------------------
-- Timer Process (FIXED)
--------------------------------------------------
process(clk, rst_n)
begin
    if rst_n = '0' then
        timer_count  <= 0;
        timer_active <= '0';
        timer_done   <= '0';
    elsif rising_edge(clk) then
        timer_done <= '0';

        if timer_start = '1' then
            timer_count  <= 0;
            timer_active <= '1';
        elsif tick = '1' and timer_active = '1' then
            if timer_count >= timer_limit then
                timer_active <= '0';
                timer_done   <= '1';
            else
                timer_count <= timer_count + 1;
            end if;
        end if;
    end if;
end process;

--------------------------------------------------
-- State Register
--------------------------------------------------
process(clk, rst_n)
begin
    if rst_n = '0' then
        state <= S_NS_G;
    elsif rising_edge(clk) then
        if emergency = '1' then
            state <= S_EMERG_ALL_RED;
        elsif night_mode = '1' then
            state <= S_NIGHT_FLASH;
        else
            state <= next_state;
        end if;
    end if;
end process;

--------------------------------------------------
-- Next State Logic + Outputs
--------------------------------------------------
process(state, timer_done, veh_ns, veh_ew, ped_req_latched)
begin
    -- defaults
    next_state <= state;
    timer_start <= '0';
    timer_limit <= 0;

    ns_g <= '0'; ns_y <= '0'; ns_r <= '0';
    ew_g <= '0'; ew_y <= '0'; ew_r <= '0';
    ped_walk <= '0'; ped_dontwalk <= '0';

    case state is

    when S_NS_G =>
        ns_g <= '1'; ew_r <= '1'; ped_dontwalk <= '1';
        timer_limit <= T_GREEN_MIN;
        timer_start <= not timer_active;

        if timer_done = '1' and (veh_ew = '1' or ped_req_latched = '1') then
            next_state <= S_NS_Y;
        end if;

    when S_NS_Y =>
        ns_y <= '1'; ew_r <= '1'; ped_dontwalk <= '1';
        timer_limit <= T_YELLOW;
        timer_start <= not timer_active;

        if timer_done = '1' then
            next_state <= S_ALL_RED1;
        end if;

    when S_ALL_RED1 =>
        ns_r <= '1'; ew_r <= '1'; ped_dontwalk <= '1';
        timer_limit <= T_ALLRED;
        timer_start <= not timer_active;

        if timer_done = '1' then
            if ped_req_latched = '1' then
                next_state <= S_PED_WALK;
            else
                next_state <= S_EW_G;
            end if;
        end if;

    when S_EW_G =>
        ew_g <= '1'; ns_r <= '1'; ped_dontwalk <= '1';
        timer_limit <= T_GREEN_MIN;
        timer_start <= not timer_active;

        if timer_done = '1' and (veh_ns = '1' or ped_req_latched = '1') then
            next_state <= S_EW_Y;
        end if;

    when S_EW_Y =>
        ew_y <= '1'; ns_r <= '1'; ped_dontwalk <= '1';
        timer_limit <= T_YELLOW;
        timer_start <= not timer_active;

        if timer_done = '1' then
            next_state <= S_ALL_RED2;
        end if;

    when S_ALL_RED2 =>
        ns_r <= '1'; ew_r <= '1'; ped_dontwalk <= '1';
        timer_limit <= T_ALLRED;
        timer_start <= not timer_active;

        if timer_done = '1' then
            if ped_req_latched = '1' then
                next_state <= S_PED_WALK;
            else
                next_state <= S_NS_G;
            end if;
        end if;

    when S_PED_WALK =>
        ns_r <= '1'; ew_r <= '1'; ped_walk <= '1';
        timer_limit <= T_WALK;
        timer_start <= not timer_active;

        if timer_done = '1' then
            next_state <= S_PED_CLEAR;
        end if;

    when S_PED_CLEAR =>
        ns_r <= '1'; ew_r <= '1'; ped_dontwalk <= '1';
        timer_limit <= T_ALLRED;
        timer_start <= not timer_active;

        if timer_done = '1' then
            next_state <= S_NS_G;
        end if;

    when S_EMERG_ALL_RED =>
        ns_r <= '1'; ew_r <= '1'; ped_dontwalk <= '1';

    when S_NIGHT_FLASH =>
        ped_dontwalk <= '1';

        if MAIN_IS_NS = 1 then
            ns_y <= tick;
            ew_r <= tick;
        else
            ew_y <= tick;
            ns_r <= tick;
        end if;

    when others =>
        next_state <= S_NS_G;

    end case;
end process;

end architecture;
