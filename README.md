//@version=5
strategy("DCA 策略", 
   overlay=true,
   max_labels_count = 500,
   pyramiding = 5,
   default_qty_type = strategy.cash,
   initial_capital = 1000,
   commission_type = strategy.commission.percent,
   commission_value = 0.07,
   default_qty_value = 100
   )

// 輸入設定
color bgray                                  = color.rgb(128, 223, 236, 81)
color WHITE                                  = color.rgb(210, 210, 210)
color red                                    = color.rgb(128, 49, 49, 67)
color white                                  = color.rgb(248, 248, 248, 14)
color olive                                  = color.olive
color blue                                   = color.rgb(23, 43, 77)
color Yellow                                 = color.rgb(255, 186, 75)
color BG                                     = color.from_gradient(close,low,high ,color.rgb(16, 194, 167),color.rgb(240, 141, 71))
string s_group                               = "策略設定"
string visual                                = "視覺效果" 
int offset                                   = 15
var int glb_dealstart_bar_time               = 0
int Length                                   = 14
float ChangePercentage                       = input.float(title='SRP %',defval=1.4,group='SRP 設定',inline = "00",step=0.1)
float core                                   = ta.vwma(hlc3, Length)  
float below                                  = input(55,title="⬇️",group='SRP 設定',inline = "00")
float above                                  = input(100,title="⬆️",group='SRP 設定',inline = "00")
bool mintpbool                               = input.bool(true,"最小利潤❓➔ ",group = s_group,inline="01")
float mintp                                  = input.float(2.0,"",step= 0.1 , group = s_group,inline="01")/100
float minSOchange                            = input.float(title='價格變動百分比', defval=2.0, step=0.1, group = s_group,inline="01")
float base                                   = input.float(title='$ 基本單位', defval=100, step=1, group = s_group,inline="01")
float safety                                 = input.float(title='DCA 倍數', defval=1.5, step=1, group = s_group,inline="01",tooltip = "它將以此值乘以部位大小")
int sonum                                    = input.int(title='DCA 次數', defval=4, step=1, group = s_group,inline="01",tooltip = "你想進行多少次DCA訂單？")+1
bool DCATYPE                                 = input.string("成交量倍增",title ="DCA 類型",options = ["成交量倍增","基本單位倍增"],group = s_group,inline="01") == "成交量倍增"


bool limit_date_range                        = input.bool(title='回測日期', defval=true, group="回測日期")
int start_time                               = input.time(defval=timestamp('11 OCT 2022 00:00 +0000'), title='開始時間', group="回測日期")
int end_time                                 = input.time(defval=timestamp('31 Dec 2080 00:00 +0000'), title='結束時間', group="回測日期")

bool show_table                              = input(true,title= "顯示表格❓",inline = "001",group = visual)
string position                              = input.string("頂部右側", options = ["中間右側","頂部中央","頂部右側","中間左側","中間中央","底部左側","底部中央","底部右側"],title = '位置',inline = "001",group = visual)
color textcolos                              = input.color(Yellow,title = "文字顏色",inline = "001",group = visual)
bool showdeals                               = input(true,"顯示交易線❓",inline = "002",group =visual )
color profitcolor                            = input.color(bgray,"",inline = "002",group =visual )  
color socolor                                = input.color(red,"",inline = "002",group =visual )  
color avgcolor                               = input.color(white,"",inline = "002",group =visual )
bool showBG                                  = input(false,"交易背景色❓",inline = "003",group =visual ) 
color bgcolor                                = input.color(blue,"",inline = "003",group =visual )  
bool showpnllabel                            = input(true,"PNL標籤❓",inline = "004",group =visual )
color pnlcolor                               = input.color(olive,"",inline = "004",group =visual ) 



string table_name                            ='SRP 策略 [ZCrypto]'
var int socounter = 0
var int dealcount = 0

// 核心計算 ---------{

in_date_range = true
if limit_date_range
    in_date_range := time >= start_time and time <= end_time
    
else
    in_date_range := true


vwma_above = core * (1 + (ChangePercentage / 100))
vwma_below = core * (1 - (ChangePercentage / 100))

//
up = ta.rma(math.max(ta.change(close), 0), 7)
down = ta.rma(-math.min(ta.change(close), 0), 7)
rsi = down == 0 ? 100 : up == 0 ? 0 : 100 - (100 / (1 + up / down))
mf = ta.mfi(hlc3, 7)
rsi_mfi = math.abs(rsi+mf/2)

long = low <= vwma_below and (rsi_mfi < below)
short = high >= vwma_above and (rsi_mfi > above)

table_position() =>
    switch position
        "中間右側" => position.middle_right
        "頂部中央" => position.top_center
        "頂部右側" => position.top_right
        "中間左側" => position.middle_left
        "中間中央" => position.middle_center
        "底部左側" => position.bottom_left
        "底部中央" => position.bottom_center
        "底部右側" => position.bottom_right      
      
var summary_table          = table.new(table_position(), 15, 33, frame_color=color.rgb(58, 58, 59, 38), frame_width=3,border_width = 2)
change_color(con)=>
    con ? color.rgb(0, 149, 77) : color.rgb(152, 46, 46)    

table_cell(table,_row,in1,in2)=>
    
    table.cell(table, 0, _row, 
     in1, 
     text_color=textcolos,
     bgcolor = color.rgb(58, 58, 59, 38),
     text_size = size.small,
     text_halign = text.align_left
     )
    table.cell(table, 1, _row, str.tostring(in2), text_color=WHITE,text_size = size.small,bgcolor =color.rgb(120, 123, 134, 38) )

//------------}
calcNextSOprice(pcnt) =>
    if strategy.position_size > 0
        strategy.position_avg_price - (math.round(pcnt / 100 * strategy.position_avg_price / syminfo.mintick)) * syminfo.mintick
    else if strategy.position_size < 0
        strategy.position_avg_price + (math.round(pcnt / 100 * strategy.position_avg_price / syminfo.mintick)) * syminfo.mintick
    else
        na
calcChnagefromlastdeal() =>
    last_deal = strategy.opentrades.entry_price(strategy.opentrades - 1)
    math.abs(((close - (last_deal)  ) /close )  * 100 )


Calcprofitpercent() =>
    math.abs(((close - strategy.position_avg_price  ) /close )  * 100 )


// def calculate(base, SO):
    
//     return so_size    
SO = base * safety
CapitalCalculation() =>
    float total = 0.0

    for x= 1 to sonum - 1 by 1
        so_size = SO * math.pow(safety, x-1)
        total+= so_size


    total + base

// plot(SO,"?111!") 
// plot(CapitalCalculation(),"?!")    
calculateSO(num)=>
    SOv = base * math.pow(safety,num)
    SOv

SOconditions()=>
    (calcChnagefromlastdeal() > minSOchange ) and (close < calcNextSOprice(minSOchange))
   

mintpclogic()=> 

   (close > (strategy.position_avg_price * (1 + mintp)))

get_timestring_from_seconds(seconds) =>
    if seconds >= 86400
        string _string = str.tostring(math.round(seconds / 86400, 1)) + ' 天'
    else if seconds >= 3600
        string _string = str.tostring(math.round(seconds / 3600, 1)) + ' 小時'
    else
        string _string = str.tostring(math.round(seconds / 60, 1)) + ' 分鐘'

get_timespan_string(start_time, end_time) =>
    _seconds_diff = (end_time - start_time) / 1000
    get_timestring_from_seconds(_seconds_diff)

Calcprofit()=>
    (( close * strategy.position_size ) - (strategy.position_avg_price * strategy.position_size))
// var label t = na , label.delete(t) , t:= label.new(bar_index,high,str.tostring(mintpclogic()))

// plot(strategy.netprofit,"?!",display = display.data_window)

PNLlabel()=>    
          
    message = ""    
    message += "PNL :  " +str.tostring(math.round(Calcprofit(), 2)) + '  ' + str.tostring(syminfo.currency) + '\n'
    message += "時間:  " +get_timespan_string(glb_dealstart_bar_time, time)+ '\n'
    message += "PNL%:  " + str.tostring(math.round(Calcprofitpercent(),2))+ " %"
    topy = high + (high* 0.04)  
    label.new(bar_index + 1,
     topy, 
     text=message,
     yloc=yloc.price, 
     size=size.normal, 
     style=label.style_label_lower_left, 
     textcolor=color.black, 
     textalign = text.align_left,
     color=pnlcolor
     ) 


var float removed = na

Opentrade()=>
    strategy.opentrades > 0 ? 1 : 0

PNLpercentage()=> ((close - strategy.position_avg_price  ) /close )  * 100 

//--------------------------------------------------------------------------
//                        # 策略輸入 #                              |
//--------------------------------------------------------------------------

// plot(strategy.opentrades,display = display.data_window,title = "OpenSZ")

if (long) and strategy.opentrades == 0 and in_date_range
    socounter := 0
    dealcount+=1
    dealcount
    glb_dealstart_bar_time      := time
    strategy.entry("多頭", strategy.long,comment = "交易 # " + str.tostring(dealcount),qty = base/close )
    alert("新的多頭交易 for {{ticker}}",freq = alert.freq_once_per_bar_close)
    removed:= strategy.netprofit / close

if (long) and  SOconditions() and strategy.opentrades > 0 and strategy.opentrades < sonum
    
    socounter+=1
    socounter
    strategy.entry('多頭' , strategy.long,qty = DCATYPE ? (strategy.position_size*safety) : calculateSO(socounter)/close,comment = "SO # "+str.tostring(socounter))
    alert("新的DCA交易 for {{ticker}}",freq = alert.freq_once_per_bar_close)
    
if strategy.position_size > 0 and short and (mintpbool ? mintpclogic() : true) and in_date_range
    strategy.close("多頭",comment = " ")
    alert("交易關閉 for {{ticker}}",freq = alert.freq_once_per_bar_close)
    if showpnllabel
        PNLlabel()








   
avg = plot(showdeals? strategy.position_avg_price : na, '平均價格',color= avgcolor, style=plot.style_linebr,editable = false)
sl = plot(showdeals ? calcNextSOprice(minSOchange): na, 'SO 變化 %', color.orange, style=plot.style_linebr,editable = false)      
tp = plot(showdeals ? strategy.position_avg_price * (1 + mintp): na, 'SO change %', color.rgb(3, 233, 245), style=plot.style_linebr,editable = false)    
   
fill(tp, avg, color =profitcolor)
fill(avg, sl, color = socolor)
bgcolor(showBG and strategy.position_size > 0 ? color.new(bgcolor,90): na)   

statusOpen()=>
    Opentrade() == 0 ? str.tostring(Opentrade()) : str.tostring(Opentrade()) + "\n" + "------" + "\n" +  str.tostring(syminfo.currency) + " :" + "$" + str.tostring(math.round(strategy.openprofit,2))+ "\n" 
         + "------" + "\n" + "% " + " :" + str.tostring(math.round(PNLpercentage(),2))+ " %"
