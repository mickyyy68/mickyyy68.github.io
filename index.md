go hard



# Barrel Shifter
```verilog
`timescale 1ns/1ps

// ---------------------------------------------------------------------------
// Barrel shifter 16-bit combinatorio
// dir_i = 0 -> shift left logico
// dir_i = 1 -> shift right (arith_i = 0 logico, arith_i = 1 aritmetico)
// ---------------------------------------------------------------------------
module barrel_shifter16 (
    input  wire [15:0] data_i,
    input  wire [3:0]  shamt_i,
    input  wire        dir_i,    // 0 = left, 1 = right
    input  wire        arith_i,  // valido solo se dir_i = 1
    output wire [15:0] data_o
);
    // Stage 0: shift di 1
    wire [15:0] s0_in  = data_i;
    wire [15:0] s0_l   = {s0_in[14:0], 1'b0};
    wire [15:0] s0_r_l = {1'b0, s0_in[15:1]};
    wire [15:0] s0_r_a = {s0_in[15], s0_in[15:1]};
    wire [15:0] s0_sel = (dir_i == 1'b0) ? s0_l :
                         (arith_i ? s0_r_a : s0_r_l);
    wire [15:0] s0_out = shamt_i[0] ? s0_sel : s0_in;

    // Stage 1: shift di 2
    wire [15:0] s1_in  = s0_out;
    wire [15:0] s1_l   = {s1_in[13:0], 2'b00};
    wire [15:0] s1_r_l = {2'b00, s1_in[15:2]};
    wire [15:0] s1_r_a = {{2{s1_in[15]}}, s1_in[15:2]};
    wire [15:0] s1_sel = (dir_i == 1'b0) ? s1_l :
                         (arith_i ? s1_r_a : s1_r_l);
    wire [15:0] s1_out = shamt_i[1] ? s1_sel : s1_in;

    // Stage 2: shift di 4
    wire [15:0] s2_in  = s1_out;
    wire [15:0] s2_l   = {s2_in[11:0], 4'b0000};
    wire [15:0] s2_r_l = {4'b0000, s2_in[15:4]};
    wire [15:0] s2_r_a = {{4{s2_in[15]}}, s2_in[15:4]};
    wire [15:0] s2_sel = (dir_i == 1'b0) ? s2_l :
                         (arith_i ? s2_r_a : s2_r_l);
    wire [15:0] s2_out = shamt_i[2] ? s2_sel : s2_in;

    // Stage 3: shift di 8
    wire [15:0] s3_in  = s2_out;
    wire [15:0] s3_l   = {s3_in[7:0], 8'h00};
    wire [15:0] s3_r_l = {8'h00, s3_in[15:8]};
    wire [15:0] s3_r_a = {{8{s3_in[15]}}, s3_in[15:8]};
    wire [15:0] s3_sel = (dir_i == 1'b0) ? s3_l :
                         (arith_i ? s3_r_a : s3_r_l);
    wire [15:0] s3_out = shamt_i[3] ? s3_sel : s3_in;

    assign data_o = s3_out;

endmodule

// ---------------------------------------------------------------------------
// Mini ALU 16-bit
// ---------------------------------------------------------------------------
module mini_alu16 (
    input  wire [15:0] a_i,
    input  wire [15:0] b_i,
    input  wire [3:0]  op_i,
    input  wire [3:0]  shamt_i,
    output reg  [15:0] res_o,
    output wire        zero_o
);

    // Codici operazione
    localparam OP_ADD = 4'b0000;
    localparam OP_SUB = 4'b0001;
    localparam OP_AND = 4'b0010;
    localparam OP_OR  = 4'b0011;
    localparam OP_XOR = 4'b0100;
    localparam OP_SLL = 4'b0101;
    localparam OP_SRL = 4'b0110;
    localparam OP_SRA = 4'b0111;

    // Uscite barrel shifter
    wire [15:0] sh_left;
    wire [15:0] sh_right_log;
    wire [15:0] sh_right_arith;

    // SLL
    barrel_shifter16 u_sh_left (
        .data_i (a_i),
        .shamt_i(shamt_i),
        .dir_i  (1'b0),     // 0 = left
        .arith_i(1'b0),     // non usato
        .data_o (sh_left)
    );

    // SRL
    barrel_shifter16 u_sh_right_log (
        .data_i (a_i),
        .shamt_i(shamt_i),
        .dir_i  (1'b1),     // 1 = right
        .arith_i(1'b0),     // logical
        .data_o (sh_right_log)
    );

    // SRA
    barrel_shifter16 u_sh_right_arith (
        .data_i (a_i),
        .shamt_i(shamt_i),
        .dir_i  (1'b1),
        .arith_i(1'b1),     // arithmetic
        .data_o (sh_right_arith)
    );

    // ALU combinatoria
    always @(*) begin
        case (op_i)
            OP_ADD: res_o = a_i + b_i;
            OP_SUB: res_o = a_i - b_i;
            OP_AND: res_o = a_i & b_i;
            OP_OR : res_o = a_i | b_i;
            OP_XOR: res_o = a_i ^ b_i;
            OP_SLL: res_o = sh_left;
            OP_SRL: res_o = sh_right_log;
            OP_SRA: res_o = sh_right_arith;
            default: res_o = 16'h0000;
        endcase
    end

    assign zero_o = (res_o == 16'h0000);

endmodule
```

# Universal Reg

```verilog
module universal_reg #(
    parameter N = 8
)(
    input clk_i,
    input reset_ni,
    input [1:0] mode_i,
    input [N-1:0] data_i,
    input ser_i,
    output reg [N-1:0] q_o
);

    reg [N-1:0] q_next;

    always @(posedge clk_i or negedge reset_ni) begin 
        if(!reset_ni) begin 
            q_o <= {N{1'b0}};
        end else begin 
            q_o <= q_next;
        end
    end 

    // combinatoria 
    always @(*) begin 
        q_next = {N{1'b0}};
        case (mode_i) 
            2'b00: q_next = q_o;
            2'b01: q_next = {q_o[N-2:0], ser_i};
            2'b10: q_next = {ser_i, q_o[N-1:1]};
            2'b11: q_next = data_i;
        endcase
    end 
endmodule
```

# Progettazione di un Controller Semaforico Intelligente aka Semaforo
```verilog
`timescale 1ns/1ps

module smart_traffic_controller (
    input  wire clk_i,
    input  wire reset_ni,
    input  wire carB_i,

    output reg  A_red_o,
    output reg  A_yellow_o,
    output reg  A_green_o,
    output reg  B_red_o,
    output reg  B_yellow_o,
    output reg  B_green_o
);


// carB_i == 1 --> avvia la sequenza.
    // dopodichè torna allo stato iniziale
// se in riposo, A verde, B rosso


    reg [1:0] state, next_state; // 8 possibili stati
    reg [1:0] timer, next_timer; // 2 bit, fino a 4 (cicli)

    // Stati
    localparam S_IDLE = 2'b00;
    localparam S_A_Y  = 2'b01;
    localparam S_B_G  = 2'b10;
    localparam S_B_Y  = 2'b11;

    // Durate (in cicli di clock)
    localparam integer A_Y_DUR = 2;
    localparam integer B_G_DUR = 3;
    localparam integer B_Y_DUR = 1;


// sequenziale per stato e timer 
    always @(posedge clk_i or negedge reset_ni) begin 
        if(!reset_ni) begin  // resetto
            state <= S_IDLE;
            timer <= 2'b0;
        end
        else begin 
            state <= next_state;
            timer <= next_timer;
        end
    end


// rete combinatoria: next_state e next_timer
    always @(*) begin 

        next_state = state;
        next_timer = timer;
    
        case (state) 
            S_IDLE: begin 
                next_timer = 2'b0;
                if (carB_i) begin 
                    next_state = S_A_Y;
                    next_timer = A_Y_DUR - 1;
                end 

            end

            S_A_Y: begin 
                if(timer == 0) begin 
                    next_state = S_B_G;
                    next_timer = B_G_DUR - 1;
                end else begin 
                    next_state = S_A_Y;
                    next_timer = timer - 1;
                end
            end

            S_B_G: begin 
                if (timer == 0) begin 
                    next_state = S_B_Y;
                    next_timer = B_Y_DUR - 1;
                end else begin 
                    next_state = S_B_G;
                    next_timer = timer - 1;
                end
            end

            S_B_Y: begin 
                if (timer == 0) begin 
                    next_state = S_IDLE;
                    next_timer = 2'b0;
                end else begin 
                    next_state = S_B_Y;
                    next_timer = timer - 1;
                end
            end

        endcase
    end

    always @(*) begin 
        A_green_o = 1'b0;
        A_yellow_o = 1'b0;
        A_red_o = 1'b0;

        B_green_o = 1'b0;
        B_yellow_o = 1'b0;
        B_red_o = 1'b0;


        case (state) 
            S_IDLE: begin 
                A_green_o = 1'b1;
                B_red_o = 1'b1;
            end

            S_A_Y: begin 
                A_yellow_o = 1'b1;
                B_red_o = 1'b1;
            end

            S_B_G: begin 
                B_green_o = 1'b1;
                A_red_o = 1'b1;
            end 

            S_B_Y: begin 
                B_yellow_o = 1'b1;
                A_red_o = 1'b1;
            end 

            default: begin 
                A_red_o = 1'b1;
                B_red_o = 1'b1;
            end 
        endcase 

    end

endmodule
```


# Mini-CPU a singolo ciclo a 8-bit
```verilog
`timescale 1ns/1ps

// Register File 4x8 - scheletro (solo interfaccia)
module register_file_4x8 (
    input  wire       clk_i,
    input  wire       reset_ni,
    input  wire [1:0] rs1_i,
    input  wire [1:0] rs2_i,
    input  wire [1:0] rd_i,
    input  wire [7:0] wdata_i,
    input  wire       we_i,
    output wire [7:0] rs1_data_o,
    output wire [7:0] rs2_data_o,
    output wire [7:0] rd_data_o
);

    // TODO: dichiarare i registri interni (4 x 8 bit)
    // TODO: implementare logica di reset
    // TODO: implementare scrittura sincrona su rd_i quando we_i = 1
    // TODO: implementare letture combinatorie su rs1_i, rs2_i, rd_i

    reg [7:0] regs [3:0];

    integer i;

    always @(posedge clk_i or negedge reset_ni) begin 
        if(!reset_ni) begin 
            for (i = 0; i < 4; i = i+1)
                regs[i] <= 8'b0;
        end 
        
        else begin 
            if(we_i) regs[rd_i] <= wdata_i;
        end
    end

    assign rs1_data_o = regs[rs1_i];
    assign rs2_data_o = regs[rs2_i];
    assign rd_data_o = regs[rd_i];

endmodule
```


# Full Adder

```verilog
module full_adder(
    input a_i,
    input b_i,
    input c_i,
    output s_o,
    output c_o
);
    assign {c_o, s_o} = a_i + b_i + c_i;
endmodule

module full_adder_3b(
    input [2:0] a_i,
    input [2:0] b_i,
    input c_i,
    output [2:0] s_o,
    output c_o
);
    wire [1:0] c_out;

    full_adder fa0 (.a_i(a_i[0]), .b_i(b_i[0]), .c_i(c_i), .s_o(s_o[0]), .c_o(c_out[0]));
    full_adder fa1 (.a_i(a_i[1]), .b_i(b_i[1]), .c_i(c_out[0]), .s_o(s_o[1]), .c_o(c_out[1]));
    full_adder fa2 (.a_i(a_i[2]), .b_i(b_i[2]), .c_i(c_out[1]), .s_o(s_o[2]), .c_o(c_o));
endmodule
```
