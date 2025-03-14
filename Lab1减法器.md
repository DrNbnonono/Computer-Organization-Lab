#### subtractor.v

```verilog
module subtractor(
    input  [31:0] operand1,  // 被减数
    input  [31:0] operand2,  // 减数
    input         cin,       // 来自低位的借位
    output [31:0] result,    // 差
    output        cout       // 向高位的借位
);
    wire [31:0] neg_operand2;  // 减数的补码（即 -operand2）
    assign neg_operand2 = ~operand2 + 1;  // 取反加1得到补码
    // 使用加法器实现减法
    assign {cout, result} = operand1 + neg_operand2 + cin;
endmodule

//这里减法的逻辑为两个补码的减法逻辑
//1.A – B 转换为加法的逻辑是 A + (-B) 
//2.因此在计算之前，首先将B转换为其补码，随后再进行相加的操作

```

#### testbench.v

```verilog
`timescale 1ns / 1ps   // 仿真单位时间为1ns，精度为1ps

module testbench;
    // Inputs
    reg [31:0] operand1;
    reg [31:0] operand2;
    reg cin;

    // Outputs
    wire [31:0] result;
    wire cout;

    // Instantiate the Unit Under Test (UUT)
    subtractor uut (
        .operand1(operand1), 
        .operand2(operand2), 
        .cin(cin), 
        .result(result), 
        .cout(cout)
    );

    initial begin
        // Initialize Inputs
        operand1 = 0;
        operand2 = 0;
        cin = 0;

        // Wait 100 ns for global reset to finish
        #100;

        // Test Case 1: 正数减正数
        operand1 = 32'd10;  // 10
        operand2 = 32'd5;   // 5
        cin = 0;            // 无借位
        #10;                // 等待10ns

        // Test Case 2: 负数减正数
        operand1 = 32'hFFFFFFF6;  // -10 (补码表示)
        operand2 = 32'd5;         // 5
        cin = 0;                  // 无借位
        #10;

        // Test Case 3: 正数减负数
        operand1 = 32'd10;        // 10
        operand2 = 32'hFFFFFFFB;  // -5 (补码表示)
        cin = 0;                  // 无借位
        #10;

        // Test Case 4: 负数减负数
        operand1 = 32'hFFFFFFF6;  // -10 (补码表示)
        operand2 = 32'hFFFFFFFB;  // -5 (补码表示)
        cin = 0;                  // 无借位
        #10;

        // Test Case 5: 带借位的减法
        operand1 = 32'd10;        // 10
        operand2 = 32'd15;        // 15
        cin = 1;                  // 有借位
        #10;

        // 结束仿真
        $stop;
    end
endmodule
```

#### **subtractor_display.v**

```verilog
module subtractor_display(
    input clk,
    input resetn,    // 低电平有效
    input input_sel, // 0:输入为被减数; 1:输入为减数
    input sw_cin,    // 来自低位的借位
    output led_cout, // 向高位的借位
    //触摸屏相关接口，不需要改
    output lcd_rst,
    output lcd_cs,
    output lcd_rs,
    output lcd_wr,
    output lcd_rd,
    inout [15:0] lcd_data_io,
    output lcd_bl_ctr,
    inout ct_int,
    inout ct_sda,
    output ct_scl,
    output ct_rstn
	);
    // 内部信号
    reg  [31:0] subtractor_operand1;
    reg  [31:0] subtractor_operand2;
    wire        subtractor_cin;
    wire [31:0] subtractor_result;
    wire        subtractor_cout;
    // 实例化减法器
    subtractor subtractor_module(
        .operand1(subtractor_operand1),
        .operand2(subtractor_operand2),
        .cin(subtractor_cin),
        .result(subtractor_result),
        .cout(subtractor_cout)
    );
    // 连接输入输出
    assign subtractor_cin = sw_cin;
    assign led_cout = subtractor_cout;
    //-----{实例化触摸屏}begin
    //此小节不需要更改
    reg         display_valid;
    reg  [39:0] display_name;//五个字符
    reg  [31:0] display_value;//数字都是32位的
    wire [5 :0] display_number;//刷新0~44，则一共需要5个
    wire        input_valid;
    wire [31:0] input_value;

    //实例化LCD屏
    lcd_module lcd_module(
        .clk            (clk           ),   //10Mhz
        .resetn         (resetn        ),

         //调用触摸屏的接口
        .display_valid  (display_valid ),
        .display_name   (display_name  ),
        .display_value  (display_value ),
        .display_number (display_number),
        .input_valid    (input_valid   ),
        .input_value    (input_value   ),

         //lcd触摸屏相关接口，不需要更改
        .lcd_rst        (lcd_rst       ),
        .lcd_cs         (lcd_cs        ),
        .lcd_rs         (lcd_rs        ),
        .lcd_wr         (lcd_wr        ),
        .lcd_rd         (lcd_rd        ),
        .lcd_data_io    (lcd_data_io   ),
        .lcd_bl_ctr     (lcd_bl_ctr    ),
        .ct_int         (ct_int        ),
        .ct_sda         (ct_sda        ),
        .ct_scl         (ct_scl        ),
        .ct_rstn        (ct_rstn       )
    ); 
	//-----{实例化触摸屏}end
    // 从触摸屏获取输入
    always @(posedge clk) 
    begin
        if (!resetn) 
        begin
            subtractor_operand1 <= 32'd0;
        end
        else if (input_valid && !input_sel) 
        begin
            subtractor_operand1 <= input_value;
        end
    end
	always @(posedge clk) 
    begin
        if (!resetn) 
        begin
            subtractor_operand2 <= 32'd0;
        end
        else if (input_valid && input_sel) 
        begin
            subtractor_operand2 <= input_value;
        end
    end
    // 输出到LCD屏
    always @(posedge clk)
    begin
        case(display_number)
            6'd1: begin
                display_valid <= 1'b1;
                display_name  <= "SUB_1"; // 被减数
                display_value <= subtractor_operand1;
            end
            6'd2: begin
                display_valid <= 1'b1;
                display_name  <= "SUB_2"; // 减数
                display_value <= subtractor_operand2;
            end
            6'd3: begin
                display_valid <= 1'b1;
                display_name  <= "RESUL"; // 结果
                display_value <= subtractor_result;
            end
            default: begin
                display_valid <= 1'b0;
                display_name  <= 40'd0;
                display_value <= 32'd0;
            end
        endcase
    end
endmodule
```

#### 添加LCD模块

#### 约束文件

```verilog
#对每个信号绑定实验向上的位置
set_property PACKAGE_PIN AC19 [get_ports clk]  #将时钟信号，绑定到AC19信号当中
set_property PACKAGE_PIN H7   [get_ports led_cout]
set_property PACKAGE_PIN Y3   [get_ports resetn]
set_property PACKAGE_PIN AC21 [get_ports input_sel]
set_property PACKAGE_PIN AD24 [get_ports sw_cin]
#给每个信号定义标准电压为3.3伏
set_property IOSTANDARD LVCMOS33 [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports led_cout]
set_property IOSTANDARD LVCMOS33 [get_ports resetn]
set_property IOSTANDARD LVCMOS33 [get_ports input_sel]
set_property IOSTANDARD LVCMOS33 [get_ports sw_cin]
#其他的内容直接复制过来
#lcd
set_property PACKAGE_PIN J25 [get_ports lcd_rst]
set_property PACKAGE_PIN H18 [get_ports lcd_cs]
set_property PACKAGE_PIN K16 [get_ports lcd_rs]
set_property PACKAGE_PIN L8 [get_ports lcd_wr]
set_property PACKAGE_PIN K8 [get_ports lcd_rd]
set_property PACKAGE_PIN J15 [get_ports lcd_bl_ctr]
set_property PACKAGE_PIN H9 [get_ports {lcd_data_io[0]}]
set_property PACKAGE_PIN K17 [get_ports {lcd_data_io[1]}]
set_property PACKAGE_PIN J20 [get_ports {lcd_data_io[2]}]
set_property PACKAGE_PIN M17 [get_ports {lcd_data_io[3]}]
set_property PACKAGE_PIN L17 [get_ports {lcd_data_io[4]}]
set_property PACKAGE_PIN L18 [get_ports {lcd_data_io[5]}]
set_property PACKAGE_PIN L15 [get_ports {lcd_data_io[6]}]
set_property PACKAGE_PIN M15 [get_ports {lcd_data_io[7]}]
set_property PACKAGE_PIN M16 [get_ports {lcd_data_io[8]}]
set_property PACKAGE_PIN L14 [get_ports {lcd_data_io[9]}]
set_property PACKAGE_PIN M14 [get_ports {lcd_data_io[10]}]
set_property PACKAGE_PIN F22 [get_ports {lcd_data_io[11]}]
set_property PACKAGE_PIN G22 [get_ports {lcd_data_io[12]}]
set_property PACKAGE_PIN G21 [get_ports {lcd_data_io[13]}]
set_property PACKAGE_PIN H24 [get_ports {lcd_data_io[14]}]
set_property PACKAGE_PIN J16 [get_ports {lcd_data_io[15]}]
set_property PACKAGE_PIN L19 [get_ports ct_int]
set_property PACKAGE_PIN J24 [get_ports ct_sda]
set_property PACKAGE_PIN H21 [get_ports ct_scl]
set_property PACKAGE_PIN G24 [get_ports ct_rstn]

set_property IOSTANDARD LVCMOS33 [get_ports lcd_rst]
set_property IOSTANDARD LVCMOS33 [get_ports lcd_cs]
set_property IOSTANDARD LVCMOS33 [get_ports lcd_rs]
set_property IOSTANDARD LVCMOS33 [get_ports lcd_wr]
set_property IOSTANDARD LVCMOS33 [get_ports lcd_rd]
set_property IOSTANDARD LVCMOS33 [get_ports lcd_bl_ctr]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[4]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[5]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[6]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[7]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[8]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[9]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[10]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[11]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[12]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[13]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[14]}]
set_property IOSTANDARD LVCMOS33 [get_ports {lcd_data_io[15]}]
set_property IOSTANDARD LVCMOS33 [get_ports ct_int]
set_property IOSTANDARD LVCMOS33 [get_ports ct_sda]
set_property IOSTANDARD LVCMOS33 [get_ports ct_scl]
set_property IOSTANDARD LVCMOS33 [get_ports ct_rstn]
```



