Connecting LCD to FPGA (Artix-7 xc7a35tcpg236-1)
This project demonstrates how to interface a 16x2 character LCD (PmodCLP) with the Digilent Nexys A7 FPGA development board using Verilog HDL. The LCD is operated in 8-bit mode, with data and command logic implemented through a custom controller and display sequence for the text "hi teja".
lcd_fpga_interface/
├── top_lcd_hello.v         # Top-level module
├── lcd_driver.v            # HD44780-compatible LCD driver
├── README.md               # Project description and setup





# top_lcd_hello.v         # Top-level module



**module top_lcd_hello(
    input  wire clk100,
    input  wire cpu_resetn,          // BTN-reset on board (active-low)
    inout  wire [7:0] lcd_d,         // JA[1-4,7-10]
    output wire       lcd_rs,        // JB1
    output wire       lcd_rw,        // JB2 (tie low)
    output wire       lcd_e          // JB3
);
    wire rst = ~cpu_resetn;

    // Tri-state not needed - LCD is write-only in this design
    assign lcd_d = lcd_d_o;
    wire [7:0] lcd_d_o;

    // Instantiate driver
    reg        wr_en = 0;
    reg        rs    = 0;
    reg [7:0]  data  = 8'h00;
    wire       ready;

    lcd_driver #(.CLK_HZ(100_000_000)) u_drv (
        .clk(clk100),
        .rst(rst),
        .wr_en(wr_en),
        .rs_in(rs),
        .data_in(data),
        .ready(ready),
        .lcd_d(lcd_d_o),
        .lcd_rs(lcd_rs),
        .lcd_rw(lcd_rw),     // always 0 inside driver
        .lcd_e(lcd_e)
    );

    // Simple ROM with init sequence followed by "hi teja"
    localparam N = 14;
    reg [7:0] rom [0:N-1];
    initial begin
        rom[ 0] = 8'h38;   // Function set: 8-bit, 2-line
        rom[ 1] = 8'h0C;   // Display ON, cursor OFF
        rom[ 2] = 8'h01;   // Clear display
        rom[ 3] = 8'h06;   // Entry mode: increment, no shift
        // Write "hi teja"
        rom[ 4] = "h";
        rom[ 5] = "i";
        rom[ 6] = " ";
        rom[ 7] = "t";
        rom[ 8] = "e";
        rom[ 9] = "j";
        rom[10] = "a";
        // Pad with blanks or nulls
        rom[11] = 8'h00;
        rom[12] = 8'h00;
        rom[13] = 8'h00;
    end

    reg [3:0] idx = 0;
    always @(posedge clk100) begin
        if (rst) begin
            idx   <= 0;
            wr_en <= 0;
        end else if (ready && !wr_en && idx < N) begin
            data  <= rom[idx];
            rs    <= (idx >= 4);   // first four are commands, rest data
            wr_en <= 1;
            idx   <= idx + 1;
        end else if (wr_en) begin
            wr_en <= 0;           // drop strobe after one clock
        end
    end
endmodule**


# lcd_driver.v            # HD44780-compatible LCD driver

**
// PmodCLP/HD44780 8-bit driver, blocking-write only (RW tied low)
module lcd_driver #(
    parameter CLK_HZ = 100_000_000
)(
    input  wire        clk,          // 100 MHz
    input  wire        rst,          // active-high synchronous reset
    input  wire        wr_en,        // strobe: 1→0 writes data_in
    input  wire        rs_in,        // 0 = command, 1 = data
    input  wire [7:0]  data_in,
    output reg         ready,        // 1 when driver can accept next byte
    output reg  [7:0]  lcd_d,
    output reg         lcd_rs,
    output reg         lcd_rw,       // hard-wired 0, but kept for completeness
    output reg         lcd_e
);

    localparam T_E_PW_NS   = 450;      // E high time (ns) 450ns min
    localparam T_CMD_US    = 50;       // generic command cycle 50µs
    localparam T_CLEAR_US  = 2000;     // clear/home  >1.52ms

    // convert times to clock counts
    localparam CNT_E_PW  = (CLK_HZ / 1_000_000_000) * T_E_PW_NS;
    localparam CNT_CMD   = (CLK_HZ / 1_000_000) * T_CMD_US;
    localparam CNT_CLEAR = (CLK_HZ / 1_000_000) * T_CLEAR_US;

    reg [31:0] cnt;
    reg [1:0]  state;    // 0-idle,1-pulse,2-wait
    reg        long_op;  // need long post-delay (clear/home)

    always @(posedge clk) begin
        if (rst) begin
            state   <= 0;
            ready   <= 1;
            lcd_e   <= 0;
            lcd_rw  <= 0;
        end else begin
            case (state)
            0: begin
                if (wr_en) begin
                    ready   <= 0;
                    lcd_d   <= data_in;
                    lcd_rs  <= rs_in;
                    lcd_e   <= 1;       // start E high pulse
                    cnt     <= CNT_E_PW;
                    long_op <= (data_in == 8'h01) || (data_in == 8'h02); // clear / home
                    state   <= 1;
                end
            end
            1: begin // finish E pulse
                if (cnt == 0) begin
                    lcd_e <= 0;
                    cnt   <= long_op ? CNT_CLEAR : CNT_CMD;
                    state <= 2;
                end else cnt <= cnt - 1;
            end
            2: begin // post-command wait
                if (cnt == 0) begin
                    ready <= 1;
                    state <= 0;
                end else cnt <= cnt - 1;
            end
            endcase
        end
    end
endmodule
**


# hardware connections are 

🔧 Hardware Setup
| PmodCLP pin     | Signal     | Nexys A7 port  | Artix‑7 pin | Notes                        |
| --------------- | ---------- | -------------- | ----------- | ---------------------------- |
| **J1‑1 (DB0)**  | lcd\_d\[0] | **JA1**        | C17         | Data bus bit 0               |
| **J1‑2 (DB1)**  | lcd\_d\[1] | **JA2**        | D18         |                              |
| **J1‑3 (DB2)**  | lcd\_d\[2] | **JA3**        | E18         |                              |
| **J1‑4 (DB3)**  | lcd\_d\[3] | **JA4**        | G17         |                              |
| **J1‑7 (DB4)**  | lcd\_d\[4] | **JA7**        | D17         |                              |
| **J1‑8 (DB5)**  | lcd\_d\[5] | **JA8**        | E17         |                              |
| **J1‑9 (DB6)**  | lcd\_d\[6] | **JA9**        | F18         |                              |
| **J1‑10 (DB7)** | lcd\_d\[7] | **JA10**       | G18         |                              |
| **J2‑1 (RS)**   | lcd\_rs    | **JB1**        | D14         | 0 = command, 1 = data        |
| **J2‑2 (R/W)**  | lcd\_rw    | **JB2**        | F16         | Tie low for write‑only       |
| **J2‑3 (E)**    | lcd\_e     | **JB3**        | G16         | Latches data on 1→0          |
| **Any GND**     | —          | JA5/11, JB5/11 | —           |                              |
| **Any VCC**     | —          | JA6/12, JB6/12 | —           | 3 V  (Rev‑B CLP accepts 3 V) |



🔍 Module Descriptions
top_lcd_hello.v
- Instantiates lcd_driver.v and feeds a sequence of characters from a local ROM.
- Sends the string "hi teja" after LCD initialization.
- Uses a simple FSM to manage timing and command/data logic.
lcd_driver.v
- Handles LCD command execution and data writes over an 8-bit bus.
- Manages enable pulse width and timing constraints.
- Recognizes long operations like clear and home and applies proper delays.








