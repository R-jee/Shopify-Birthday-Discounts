# Shopify-Birthday-Discounts
Line Items 1.7 [OD] Welcome and Birthday Discounts

<pre>
  # Customer Specific Discounts 15/11/2022 [GR]
  # This script applies 
  # --Base discounts: Birthday and Welcome
  # --Loyalty discounts: Silver or Gold
  # --Staff discounts: percentage set in tag, limited to a yearly limit
  # _____________________
  # Client settings
</pre>

$MEMBER_BIRTHDAY_DISCOUNT = 10
$MEMBER_BIRTHDAY_TAG_CREATED = 'birthday-discount::created'
$MEMBER_BIRTHDAY_TAG_CLAIMED = 'birthday-discount::claimed'
$MEMBER_BIRTHDAY_TAG_EXPIRED = 'birthday-discount::expired'

$MEMBER_WELCOME_DISCOUNT = 10
$MEMBER_WELCOME_TAG_CREATED = 'welcome-discount::created'
$MEMBER_WELCOME_TAG_CLAIMED = 'welcome-discount::claimed'
$MEMBER_WELCOME_TAG_EXPIRED = 'welcome-discount::expired'

$MEMBER_STAFF_DISCOUNT_TAG_PREFIX = 'discount::'
$MEMBER_ANNUAL_LIMIT_TAG_PREFIX = 'annual-limit::'
$MEMBER_ANNUAL_EXPENSE_TAG_PREFIX = 'annual-expense::'

$MEMBER_LOYALTY_TAG_PREFIX = 'Membership::Tier::'

$MEMBER_LOYALTY_GOLD_PERCENTAGE = 10
$MEMBER_LOYALTY_SILVER_PERCENTAGE = 5
# END OF CLIENT SETTINGS
# ______________________________________
# DO NOT EDIT BELOW THIS LINE

def get_member(cart)
  member = {
    birthday: {
      percentage: $MEMBER_BIRTHDAY_DISCOUNT,
      created: false,
      claimed: false,
      expired: false,
      qualifies: false
    },
    welcome: {
      percentage: $MEMBER_WELCOME_DISCOUNT,
      created: false,
      claimed: false,
      expired: false,
      qualifies: false
    },
    loyalty: {
      percentage: nil,
      qualifies: false
    },
    staff: {
      percentage: 0,
      qualifies: false,
      current_total: Money.zero,
      limit: Money.zero,
      total_left: Money.zero
    }
  }
  if cart.customer 
    cart.customer.tags.each do | tag |
      if tag == $MEMBER_BIRTHDAY_TAG_CREATED
        member[:birthday][:created] = true
        next
      end
      if tag == $MEMBER_BIRTHDAY_TAG_CLAIMED
        member[:birthday][:claimed] = true
        next
      end
      if tag == $MEMBER_BIRTHDAY_TAG_EXPIRED
        member[:birthday][:expired] = true
        next
      end
      
      if tag == $MEMBER_WELCOME_TAG_CREATED
        member[:welcome][:created] = true
        next
      end
      if tag == $MEMBER_WELCOME_TAG_CLAIMED
        member[:welcome][:claimed] = true
        next
      end
      if tag == $MEMBER_WELCOME_TAG_EXPIRED
        member[:welcome][:expired] = true
        next
      end

      if tag.include? $MEMBER_LOYALTY_TAG_PREFIX
        tier_name = tag.split('::')[2]
        
        percent_discount = 0
        case tier_name
          when "Gold"
            percent_discount = $MEMBER_LOYALTY_GOLD_PERCENTAGE
            member[:loyalty][:qualifies] = true
          when "Silver"
            percent_discount = $MEMBER_LOYALTY_SILVER_PERCENTAGE
            member[:loyalty][:qualifies] = true
          else
            percent_discount = 0
        end
        member[:loyalty][:percentage] = percent_discount.to_f
        next
      end

      if tag.include? $MEMBER_STAFF_DISCOUNT_TAG_PREFIX 
        tag_split = tag.split('::')
        discount = tag_split[-1]
        discount_type = discount[-1]
   
        if discount_type == '%'
          member[:staff][:percentage] = discount.sub("%","").to_f
        end
        next
      end

      if tag.include? $MEMBER_ANNUAL_EXPENSE_TAG_PREFIX
        tag_split = tag.split('::')
        annual_expense = Money.new(cents: tag_split[-1].to_f*100)
        member[:staff][:current_total] = annual_expense
        next
      end

      if tag.include? $MEMBER_ANNUAL_LIMIT_TAG_PREFIX
        tag_split = tag.split('::')
        annual_limit = Money.new(cents: tag_split[-1].to_f*100)
        member[:staff][:limit] = annual_limit
        next
      end

    end

    if member[:staff][:current_total] and member[:staff][:limit] and member[:staff][:percentage]
      member[:staff][:total_left] = member[:staff][:limit] - member[:staff][:current_total]
      if member[:staff][:total_left] > Money.zero and member[:staff][:percentage] > 0
        member[:staff][:qualifies] = true
      end
    end 
    if member[:birthday][:created] and !member[:birthday][:claimed] and !member[:birthday][:expired]
      member[:birthday][:qualifies] = true
    end 
    if member[:welcome][:created] and !member[:welcome][:claimed] and !member[:welcome][:expired]
      member[:welcome][:qualifies] = true
    end 
    
  end
  return member
end

def add_line_item_props(line_item,item_info)
  old_properties = line_item.properties
  merged_properties = old_properties
  merged_properties = merged_properties.merge(item_info)
  line_item.change_properties(merged_properties, {message: 'Adding discounted price'})
end


class CustomerStackedDiscountsCampaign

  def initialize()
    @customer = Input.cart.customer
  end

  def run(cart)

    member = get_member(cart)
    member_discount = {
      threshold: Money.zero,
      pre_threshold_percentage: 0,
      post_threshold_percentage: 0,
      post_line_property: ''
    }

    # BASE DISCOUNT
    if member[:welcome][:qualifies]
      if member[:welcome][:percentage] > member_discount[:post_threshold_percentage]
        member_discount[:post_threshold_percentage] = member[:welcome][:percentage]
        member_discount[:post_line_property] = 'welcome'
        puts 'apply welcome discount'
      end
    end
    
    # BASE DISCOUNT - take highest discount from base qualifiers
    if member[:birthday][:qualifies]
      if member[:birthday][:percentage] > member_discount[:post_threshold_percentage]
        member_discount[:post_threshold_percentage] = member[:birthday][:percentage]
        member_discount[:post_line_property] = 'birthday'
        puts 'apply birthday discount'
      end
    end
    
    # LOYALTY DISCOUNT - add loyalty discount to base discount
    if member[:loyalty][:qualifies]
      if member[:loyalty][:percentage] > member_discount[:post_threshold_percentage]
        member_discount[:post_threshold_percentage] = member[:loyalty][:percentage]
        puts 'apply loyalty discount'
      end
    end
    
    # STAFF DISCOUNT - set pre threshold discount and post
    if member[:staff][:qualifies]
      member_discount[:pre_threshold_percentage] = member[:staff][:percentage]
      member_discount[:threshold] = member[:staff][:total_left]
      puts 'apply staff discount'
    end
    
    # If nothing to discount by return out
    if member_discount[:pre_threshold_percentage] == 0 and member_discount[:post_threshold_percentage] == 0
      return
    else
      puts 'threshold:'
      puts member_discount[:threshold]
      puts 'pre threshold percentage:'
      puts member_discount[:pre_threshold_percentage]
      puts 'post threshold percentage:'
      puts member_discount[:post_threshold_percentage]
      puts '11'
    end

    member_discount_percentage = member_discount[:pre_threshold_percentage]
    line_item_property = ''
    if member_discount[:threshold] > Money.zero
      threshold_met = false
      
      cart.line_items.each do |line_item|
        puts member_discount[:threshold]
        
        itemPrice = line_item.line_price
        standardDiscount = Money.zero
        if line_item.variant.compare_at_price
         if line_item.variant.compare_at_price * line_item.quantity > line_item.line_price
           itemPrice = line_item.variant.compare_at_price * line_item.quantity
           standardDiscount = (line_item.variant.compare_at_price * line_item.quantity) - line_item.line_price
          end
        end
        
        # If remaining threshold is 0 we stop applying staff discount and change to post threshold percentage
        if member_discount[:threshold] <= Money.zero
          threshold_met = true
          member_discount_percentage = member_discount[:post_threshold_percentage]
          line_item_property = member_discount[:post_line_property]
        end
        
        discount_amount = itemPrice * (member_discount_percentage / 100)
        if (standardDiscount > discount_amount)
          next
        end
        
        puts "discount_amount"
        puts discount_amount

        
        discount_message = member_discount_percentage.to_s.concat("% Off")
        
        discounted_price = itemPrice - discount_amount
    
        # If current line item pushes customer over threshold limit, don't apply staff discount
        if discounted_price >= member_discount[:threshold]
          puts 'over threshold'
          discount_amount = line_item.line_price * (member_discount[:post_threshold_percentage] / 100)
          discount_message = member_discount[:post_threshold_percentage].to_s.concat("% Off")
          discounted_price = line_item.line_price - discount_amount
          line_item_property = member_discount[:post_line_property]
        end

        if discounted_price < line_item.line_price
          line_item.change_line_price(discounted_price, { message: discount_message })
          member_discount[:threshold] = member_discount[:threshold] - discounted_price
          puts 'line_item_property:'
          puts line_item_property
          if line_item_property.size > 0
            add_line_item_props(line_item,{"_discount_type"  => line_item_property } )
          end
        end
      end
    else

      line_item_property = member_discount[:post_line_property]
      # When no staff discount - apply any loyalty or base discounts only (post_threshold_percentage)
      cart.line_items.each do |line_item|

      itemPrice = line_item.line_price
      standardDiscount = Money.zero
      if line_item.variant.compare_at_price
       if line_item.variant.compare_at_price* line_item.quantity  > line_item.line_price
         itemPrice = line_item.variant.compare_at_price * line_item.quantity 
         standardDiscount = (line_item.variant.compare_at_price * line_item.quantity)  - line_item.line_price
        end
      end
        
        discount_amount = itemPrice * (member_discount[:post_threshold_percentage] / 100)
        if (standardDiscount > discount_amount)
          next
        end
        discount_message = member_discount[:post_threshold_percentage].to_s.concat("% Off")
        discounted_price = itemPrice - discount_amount
 
        if discounted_price < line_item.line_price
          line_item.change_line_price(discounted_price, { message: discount_message })
          if line_item_property.size > 0
            add_line_item_props(line_item,{"_discount_type"  => line_item_property } )
          end
        end
        
      end
    end

  end
end


class RejectDiscountOnOrderWithSaleItems

  def initialize()
    @is_reduced = 0
  end
  
  def run(cart)
    if Input.cart.discount_code.nil?
      return
    end
    
    cart.line_items.each do |line_item|
     if line_item.variant.compare_at_price
      if line_item.variant.compare_at_price > line_item.line_price
       @is_reduced = 1
      end
     end
    end
    
    if @is_reduced == 0
      return
    end
    
    Input.cart.discount_code.reject( message: "Discount codes can not be used on sale items.")
  end
end

CAMPAIGNS = [
  CustomerStackedDiscountsCampaign.new(),
  RejectDiscountOnOrderWithSaleItems.new()
]

CAMPAIGNS.each do |campaign|
  campaign.run(Input.cart)
end

Output.cart = Input.cart

