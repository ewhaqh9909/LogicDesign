#09 적외선 컨트롤러
리모컨 송신은 일종의 통신이므로 그에 맞는 통신 규약에 따라야 한다.
1) Leader Code : 프레임의 모드 선택 
2) Custom Code : 특정 회사를 나타냄 
3) Data Code : 송신 데이터 (데이터 확인 위해 보수 신호도 보냄)

이번에 짜는 코드의 큰 틀은 Leader code는  9ms동안의 2'b11상태가 유지되고 4.5ms의 2'b00상태가 유지된다면 다음 상태로 넘어갈 수 있게 되고 datacode상태에서도 cnt32가 6'd32이면서 cnt_l가 1000이상일때 complete로 넘어갈 수 있게 되면서 출력값을 내게 되는 것이다.

moduler ir_rx에서는 코드를 순서대로 해석해보면...

IDLE는 처음상태를 나타내고  2'b00라고  parameter 로 명명한 것처럼 LEADCODE와 DATACODE와 COMPLETE를 할당해줬다.  
이 모듈에서 리셋상황일떄 2'b00으로 그게 아니라면  seq_rx <= {seq_rx[0], ir_rx};으로 현재의 seq_rx는 이전의 seq_rx[0], ir_rx로 밀린다는 것을 의미하게 된다. 그래서 그래프가  01,11,10,00으로 나타나게 된다.

여기에서 리셋상태가 아니라면 각 seq_rx에 2'b00이면 cnt_l이 이전 상태의 cnt_l에서 1이 더해진 상황이 되고 
seq_rx에 2'b01이면 cnt_l이 0, cnt_h이 0으로  2'b11 cnt_h이 이전 상태의 cnt_h에서 1 이 더해진 상황이 된다.

그리고  state상태를 나눠서 상황에 따라 결과를 다르게 해줬다. 앞서 parameter로 설정한 것들을 각각 IDLE일때는 cnt32가 0이되고 state 는 leadcode가 된다. cnt_h가 leader code의 9ms동안의 2'b11상태가 되도록 하는 것이므로 어느정도 차이를 두기위해 8500이상일때와  cnt_l이 leader code의 4.5ms의 00상태를 유지하는 상태로 어느정도 차이를 두고 4000이상일때 state가 datacode로 넘어가게 된다. 그게 아니라면 계속 leadcode 인 것이다. 
이후에 datacode는  seq_rx가 01일떄는  이전 상태의 cnt32에서 1 이 더해진 상황이 된다. 그게 아니라면 cnt32는 계속 유지된다. 그리고  cnt32가 32이상이고 cnt_l가 (1.69ms이상 유지될때 임으로) 어느정도 차이를 두어서 1000이상일때   state는 complete 이 되고 그게 아니라면 계속 datacode 로 유지되게 된다.

 또한 위의 datacode상태에서 cnt_l이 1000이상이라면 출력 data의 시작을 1으로 / 1000이상이 아니라면 0이 된다. 그리고 위에서 complete상태가 되었을때는  출력   data가 o_data로 진짜 output이 되게 된다.
 ![](wave09.png)

```verilog

module	nco(	
		o_gen_clk,
		i_nco_num,
		clk,
		rst_n);	
		

output		o_gen_clk	;	// 1Hz CLK

input	[31:0]	i_nco_num	;
input		clk		;	// 50Mhz CLK
input		rst_n		;

reg	[31:0]	cnt		;
reg		o_gen_clk	;

always @(posedge clk or negedge rst_n) begin
	if(rst_n == 1'b0) begin
		cnt		<= 32'd0;
		o_gen_clk	<= 1'd0	;
	end else begin
		if(cnt >= i_nco_num/2-1) begin
			cnt 	<= 32'd0;
			o_gen_clk	<= ~o_gen_clk;
		end else begin
			cnt <= cnt + 1'b1;
		end
	end
end

endmodule

//	--------------------------------------------------
//	Flexible Numerical Display Decoder
//	--------------------------------------------------
module	fnd_dec(
		o_seg,
		i_num);

output	[6:0]	o_seg		;	// {o_seg_a, o_seg_b, ... , o_seg_g}

input	[3:0]	i_num		;
reg	[6:0]	o_seg		;
//making
always @(i_num) begin 
 	case(i_num) 
 		4'd0:	o_seg = 7'b111_1110; 
 		4'd1:	o_seg = 7'b011_0000; 
 		4'd2:	o_seg = 7'b110_1101; 
 		4'd3:	o_seg = 7'b111_1001; 
 		4'd4:	o_seg = 7'b011_0011; 
 		4'd5:	o_seg = 7'b101_1011; 
 		4'd6:	o_seg = 7'b101_1111; 
 		4'd7:	o_seg = 7'b111_0000; 
 		4'd8:	o_seg = 7'b111_1111; 
 		4'd9:	o_seg = 7'b111_0011; 
 		4'd10:	o_seg = 7'b111_0111; 
 		4'd11:	o_seg = 7'b001_1111; 
 		4'd12:	o_seg = 7'b100_1110; 
 		4'd13:	o_seg = 7'b011_1101; 
 		4'd14:	o_seg = 7'b100_1111; 
 		4'd15:	o_seg = 7'b100_0111; 
		default:o_seg = 7'b000_0000; 
	endcase 
end


endmodule

//	--------------------------------------------------
//	0~59 --> 2 Separated Segments
//	--------------------------------------------------
module	double_fig_sep(
		o_left,
		o_right,
		i_double_fig);

output	[3:0]	o_left		;
output	[3:0]	o_right		;

input	[5:0]	i_double_fig	;

assign		o_left	= i_double_fig / 10	;
assign		o_right	= i_double_fig % 10	;

endmodule

//	--------------------------------------------------
//	0~59 --> 2 Separated Segments
//	--------------------------------------------------
module	led_disp(
		o_seg,
		o_seg_dp,
		o_seg_enb,
		i_six_digit_seg,
		i_six_dp,
		clk,
		rst_n);

output	[5:0]	o_seg_enb		;
output		o_seg_dp		;
output	[6:0]	o_seg			;

input	[41:0]	i_six_digit_seg		;
input	[5:0]	i_six_dp		;
input		clk			;
input		rst_n			;

wire		gen_clk		;

nco		u_nco(
		.o_gen_clk	( gen_clk	),
		.i_nco_num	( 32'd5000	),
		.clk		( clk		),
		.rst_n		( rst_n		));


reg	[3:0]	cnt_common_node	;

always @(posedge gen_clk or negedge rst_n) begin
	if(rst_n == 1'b0) begin
		cnt_common_node <= 4'd0;
	end else begin
		if(cnt_common_node >= 4'd5) begin
			cnt_common_node <= 4'd0;
		end else begin
			cnt_common_node <= cnt_common_node + 1'b1;
		end
	end
end

reg	[5:0]	o_seg_enb		;

always @(cnt_common_node) begin
	case (cnt_common_node)
		4'd0:	o_seg_enb = 6'b111110;
		4'd1:	o_seg_enb = 6'b111101;
		4'd2:	o_seg_enb = 6'b111011;
		4'd3:	o_seg_enb = 6'b110111;
		4'd4:	o_seg_enb = 6'b101111;
		4'd5:	o_seg_enb = 6'b011111;
		default:o_seg_enb = 6'b111111;
	endcase
end

reg		o_seg_dp		;

always @(cnt_common_node) begin
	case (cnt_common_node)
		4'd0:	o_seg_dp = i_six_dp[0];
		4'd1:	o_seg_dp = i_six_dp[1];
		4'd2:	o_seg_dp = i_six_dp[2];
		4'd3:	o_seg_dp = i_six_dp[3];
		4'd4:	o_seg_dp = i_six_dp[4];
		4'd5:	o_seg_dp = i_six_dp[5];
		default:o_seg_dp = 1'b0;
	endcase
end

reg	[6:0]	o_seg			;

always @(cnt_common_node) begin
	case (cnt_common_node)
		4'd0:	o_seg = i_six_digit_seg[6:0];
		4'd1:	o_seg = i_six_digit_seg[13:7];
		4'd2:	o_seg = i_six_digit_seg[20:14];
		4'd3:	o_seg = i_six_digit_seg[27:21];
		4'd4:	o_seg = i_six_digit_seg[34:28];
		4'd5:	o_seg = i_six_digit_seg[41:35];
		default:o_seg = 7'b111_1110; // 0 display
	endcase
end

endmodule

//	--------------------------------------------------
//	IR Rx Module: Note : Inverted IR Rx Signal
//	--------------------------------------------------
module	ir_rx(	
		o_data,
		i_ir_rxb,
		clk,
		rst_n);

output	[31:0]	o_data		;

input		i_ir_rxb	;
input		clk		;
input		rst_n		;

parameter	IDLE		= 2'b00	;
parameter	LEADCODE	= 2'b01	;	// 9ms high 4.5ms low
parameter	DATACODE	= 2'b10	;	// Custom & Data Code
parameter	COMPLETE	= 2'b11	;	// 32-bit data

//		1M Clock = 1 us Reference Time
wire		clk_1M				;
nco		u_nco(
		.o_gen_clk	( clk_1M	),
		.i_nco_num	( 32'd50	),
		.clk		( clk		),
		.rst_n		( rst_n		));

//		Sequential Rx Bits

wire		ir_rx		;
assign		ir_rx = ~i_ir_rxb;

reg	[1:0]	seq_rx				;
always @(posedge clk_1M or negedge rst_n) begin
	if(rst_n == 1'b0) begin
		seq_rx <= 2'b00;
	end else begin
		seq_rx <= {seq_rx[0], ir_rx};
	end
end

//		Count Signal Polarity (High & Low)
reg	[15:0]	cnt_h		;
reg	[15:0]	cnt_l		;
always @(posedge clk_1M or negedge rst_n) begin
	if(rst_n == 1'b0) begin
		cnt_h <= 16'd0;
		cnt_l <= 16'd0;
	end else begin
		case(seq_rx)
			2'b00	: cnt_l <= cnt_l + 1;
			2'b01	: begin
				cnt_l <= 16'd0;
				cnt_h <= 16'd0;
			end
			2'b11	: cnt_h <= cnt_h + 1;
		endcase
	end
end

//		State Machine
reg	[1:0]	state		;
reg	[5:0]	cnt32		;
always @(posedge clk_1M or negedge rst_n) begin
	if(rst_n == 1'b0) begin
		state <= IDLE;
		cnt32 <= 6'd0;
	end else begin
		case (state)
			IDLE: begin
				state <= LEADCODE;
				cnt32 <= 6'd0;
			end
			LEADCODE: begin
				if (cnt_h >= 8500 && cnt_l >= 4000) begin
					state <= DATACODE;
				end else begin
					state <= LEADCODE;
				end
			end
			DATACODE: begin
				if (seq_rx == 2'b01) begin
					cnt32 <= cnt32 + 1;
				end else begin
					cnt32 <= cnt32;
				end
				if (cnt32 >= 6'd32 && cnt_l >= 1000) begin
					state <= COMPLETE;
				end else begin
					state <= DATACODE;
				end
			end
			COMPLETE: state <= IDLE;
		endcase
	end
end

//		32bit Custom & Data Code
reg	[31:0]	data		;
reg	[31:0]	o_data		;
always @(posedge clk_1M or negedge rst_n) begin
	if(rst_n == 1'b0) begin
		data <= 32'd0;
	end else begin
		case (state)
			DATACODE: begin
				if (cnt_l >= 1000) begin
					data[32-cnt32] <= 1'b1;
				end else begin
					data[32-cnt32] <= 1'b0;
				end
			end
			COMPLETE: o_data <= data;
		endcase
	end
end


endmodule

//	--------------------------------------------------
//	Top Module
//	--------------------------------------------------
module	top(
		o_seg_enb,
		o_seg_dp,
		o_seg,
		i_ir_rxb,
		clk,
		rst_n);

output	[5:0]	o_seg_enb	;
output		o_seg_dp	;
output	[6:0]	o_seg		;

input		i_ir_rxb	;
input		clk		;
input		rst_n		;



wire	[31:0]	o_data		;




ir_rx			ir_rx_u_ir(	
					.o_data(o_data),
					.i_ir_rxb(i_ir_rxb),
					.clk(clk),
					.rst_n(rst_n));

wire	[6:0]	o_seg_0		;
wire	[6:0]	o_seg_1		;
wire	[6:0]	o_seg_2		;
wire	[6:0]	o_seg_3		;
wire	[6:0]	o_seg_4		;
wire	[6:0]	o_seg_5		;



fnd_dec			fnd_dec_u0(
					.o_seg(o_seg_0),
					.i_num(o_data[3:0]));

fnd_dec			fnd_dec_u1(
					.o_seg(o_seg_1),
					.i_num(o_data[7:4]));

fnd_dec			fnd_dec_u2(
					.o_seg(o_seg_2),
					.i_num(o_data[11:8]));

fnd_dec			fnd_dec_u3(
					.o_seg(o_seg_3),
					.i_num(o_data[15:12]));

fnd_dec			fnd_dec_u4(
					.o_seg(o_seg_4),
					.i_num(o_data[19:16]));

fnd_dec			fnd_dec_u5(
					.o_seg(o_seg_5),
					.i_num(o_data[23:20]));

wire	[41:0]	i_six_digit_seg	;
assign	i_six_digit_seg = { o_seg_5,o_seg_4,o_seg_3,o_seg_2, o_seg_1, o_seg_0};

led_disp		led_disp_u(
					.o_seg(o_seg),
					.o_seg_dp(o_seg_dp),
					.o_seg_enb(o_seg_enb),
					.i_six_digit_seg(i_six_digit_seg),
					.i_six_dp(6'h0),
					.clk(clk),
					.rst_n(rst_n));

endmodule
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjgzNzE0MDc5XX0=
-->