## `fix(decimal): INT_MIN 크기를 unsigned 범위에서 계산`

diff --git a/src/ft_number.c b/src/ft_number.c
index 1b5bf74..b47df9c 100644
--- a/src/ft_number.c
+++ b/src/ft_number.c
@@ -41,7 +41,8 @@ int	ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number)
 
 	value = (long)number;
 	if (value < 0)
-		return (ft_write_decimal(ctx, fmt, "-", (unsigned long)(-value)));
+		return (ft_write_decimal(ctx, fmt, "-",
+				(unsigned long)(-(value + 1)) + 1));
 	prefix = "";
 	if (fmt->flags & FT_FLAG_PLUS)
 		prefix = "+";
